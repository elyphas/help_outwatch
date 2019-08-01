# help_outwatch

Handler

`Handler` is an invariant Functor
If you want to go from `Handler[A]` to `Handler[B]` use `imap`.  which means you need to provide two functions, one from A => B and one from B => A

and has a referential transparent API

Because if you think about it, how would a function `A => B` be enough, now you have a new `Handler` that can emit items of type `B` using the function `A => B`, but now when you want to pipe new elements of type `B` into the `Handler` it can’t translate them back to `A`. This means that `map` on `Handler` returns an `Observable` where you can’t pipe any elements back into it, but you can still get a new handler by using `imap` and giving the `Handler` both functions.
Here are the cats docs on `Invariant` functors btw (: https://typelevel.org/cats/typeclasses/invariant.html


radio button group with a handler

```\nHandler.create[Int].flatMap { radioHandler =>\n   div(\n            input(tpe := \"radio\", name := \"test_radio\", onChange(1) --> radioHandler),\n            input(tpe := \"radio\", name := \"test_radio\", onChange(2) --> radioHandler),\n            input(tpe := \"radio\", name := \"test_radio\", onChange(3) --> radioHandler),\n            div(\"Selected: \", child <-- radioHandler)\n          )\n}\n```


@LukaJCB one of the things i noticed as i've been working through the examples is that in the newer version it looks like the patterns changes from `val  textValues = createStringHandler()` to `val textValues = Handler.create[String].unsafeRunSync()` is that right? when is the right time to put the .unsafeRunSync on there?
It’d be better to wrap the whole thing in a for-expression instead 
I.e.\n```scala\nfor {\n  additions <- Handler.create(0)\n  subtractions <- Handler.create(0)\n  state = Observable.merge(additions, subtractions)\n      .scan(0)((acc,cur) => acc + cur)\n} yield …\n```

and then the .unsafeRunSync or runAsync stuff and whatnot would only be at the very end?
suggest using the for { } yield pattern for components too




so basically you'd rewrite the \"textFieldComponent\" to be more like: \n```\n    def textFieldComponent(outputEvents: Sink[String]) = for {\n      textValues <- Handler.create[String](\"\")\n      disabledValues = textValues.map(_.length < 4)\n    } yield div(\n      label(\"Todo: \"),\n      input(onInput.value --> textValues),\n      button(onClick(textValues) --> outputEvents, disabled <-- disabledValues,\"Submit\")\n    )\n```\nis that right? run it in `val root = div(textfieldComponent(outputEvents).unsafeRunSync` ?




It should be \n```scala\ndef textFieldComponent(outputEvents: Sink[String]) = for {\n      textValues <- Handler.create[String](\"\")\n      disabledValues = textValues.map(_.length < 4)\n      result <- div(\n        label(\"Todo: \"),\n        input(onInput.value --> textValues),\n        button(onClick(textValues) --> outputEvents, disabled <-- disabledValues,\"Submit\")\n      )\n} yield result\n```
Unfortunately Scala’s for-comprehension don’t allow you to just return a monadic value in the `yield`block
ok, and then to run it, you'd do the following? \n```\nval mysink = Handler.create[String](\"\")  //should .unsafeRunSync() be added here?\nOutwatch.renderInto(\"#app\", textFieldComponent(mysink))\n```
Nope, just use another for-comprehension :)
```scala\nfor {\n  mySink <- Handler.create(“”)\n  _ OutWatch.renderIntl(“#app”, textFieldComponent(mySink))\n} yield ()\n```








here's what i have so far (and only get a blank screen):\n```\n    def textFieldComponent(outputEvents: Sink[String]) = for {\n      textValues <- Handler.create[String](\"\")\n      disabledValues = textValues.map(_.length < 4)\n      result <- div(\n        label(\"Todo: \"),\n        input(onInput.value --> textValues),\n        button(onClick(textValues) --> outputEvents, disabled <-- disabledValues,\"Submit\")\n      )\n    } yield result\n\n    for {\n      mySink <- Handler.create(\"\")\n      _ <- OutWatch.renderInto(\"#app\",textFieldComponent(mySink))\n    } yield ()\n    \n```














for what it's worth, here's the final working  todo list app:\n```\n    def textFieldComponent(outputEvents: Sink[String]) = for {\n      textValues <- Handler.create[String](\"\")\n      disabledValues = textValues.map(_.length < 4)\n      result <- div(\n        label(\"Todo: \"),\n        input(onInput.value --> textValues),\n        button(onClick(textValues) --> outputEvents, disabled <-- disabledValues,\"Submit\")\n      )\n    } yield result\n\n    def todoItemComponent(todo: String, deleteEvents: Sink[String]) = li(\n      span(todo),\n      button(onClick(todo) --> deleteEvents, \"Delete\")\n    )\n\n    val todoComponent = {\n      def addToList(todo: String) = (l: List[String]) => (l :+ todo).toList\n      def removeFromList(todo: String) = (l: List[String]) => l.filterNot(_ == todo)\n      for {\n        addEvents <- Handler.create[String]\n        deleteEvents <- Handler.create[String]\n        additions = addEvents.map(addToList _)\n        deletions = deleteEvents.map(removeFromList _)\n        merged = Observable.merge(additions,deletions).scan(List.empty[String])((acc,fn) => fn.apply(acc))\n        listViews = merged.map(_.map(todo => todoItemComponent(todo,deleteEvents)))\n        result = div(textFieldComponent(addEvents), ul(children <-- listViews))\n      } yield result\n    }\n\n    todoComponent.flatMap(v => OutWatch.renderInto(\"#app\",v)).unsafeRunSync()\n```

I think you can remove the last flatMap by turning the last line of the for-expression into `result <- div(…)` :)



hmm, shouldn't clearEvents be a Handler, not an observable?
Yes it should be contained to the component

so is this a more idiomatic way of doing things?\n```\n    def textFieldComponent(outputEvents: Sink[String]) = for {\n      rawTextValues <- Handler.create[String](\"\")\n      candidate <- Handler.create[String](\"\")\n      result <- div(\n        input(onInput.value --> rawTextValues, onKeyPress.collect{case e if e.keyCode == KeyCode.Enter => e}.value --> candidate),\n        button(onClick(rawTextValues) --> candidate, \"Submit\")\n      )\n      _ <- outputEvents <-- candidate\n    } yield result\n```
that component lets me hit enter or click the submit button
and then this is the updated version that also clears the text field out after either enter key or button is pressed\n```\n    def textFieldComponent(outputEvents: Sink[String]) = for {\n      rawTextValues <- Handler.create[String](\"\")\n      candidate <- Handler.create[String](\"\")\n      clearEvents = candidate.map(_ => \"\")\n      result <- div(\n        input(onInput.value --> rawTextValues, onKeyPress.collect{case e if e.keyCode == KeyCode.Enter => e}.value --> candidate, value <-- clearEvents),\n        button(onClick(rawTextValues) --> candidate, \"Submit\")\n      )\n      _ <- outputEvents <-- candidate\n    } yield result\n```


@fdietze got it, and thanks for the encouragement! So, long day's journey into night (after hitting my head against the wall for quite sometime), I came up with the following as a functional todo list\n```\n    def textFieldComponent(outputEvents: Sink[String]) = for {\n      rawTextValues <- Handler.create[String](\"\")\n      candidate <- Handler.create[String](\"\")\n      clearEvents = candidate.map(_ => \"\") //always emits an empty string whenever the candidate string updates\n      disabledFlag = rawTextValues.map(s => s.length < 4) //a flag that when true should disable the button AND also not allow submitting via enter key\n      result <- div(\n        input(onInput.value --> rawTextValues, onKeyPress.collect{case e if e.keyCode == KeyCode.Enter => e}.value.filter(_.length >= 4) --> candidate, value <-- clearEvents),\n        button(onClick(rawTextValues).filter(_.nonEmpty) --> candidate, \"Submit\", disabled <-- disabledFlag)\n      )\n      _ <- outputEvents <-- candidate //send output to parent component\n      _ <- rawTextValues <-- clearEvents //clear the input component\n    } yield result\n\n    def todoItemComponent(todo: String, deleteEvents: Sink[String]) = for(\n      result <- li(\n        span(todo),\n        button(onClick(todo) --> deleteEvents, \"Delete\")\n      )\n    ) yield result\n\n    val todoComponent = {\n      def addToList(todo: String) = (l: List[String]) => (l :+ todo).toList\n      def removeFromList(todo: String) = (l: List[String]) => l.filterNot(_ == todo)\n      for {\n        addEvents <- Handler.create[String]\n        deleteEvents <- Handler.create[String]\n        additions = addEvents.map(addToList _)\n        deletions = deleteEvents.map(removeFromList _)\n        merged = Observable.merge(additions,deletions).scan(List.empty[String])((acc,fn) => fn.apply(acc))\n        listViews = merged.map(_.map(todo => todoItemComponent(todo,deleteEvents)))\n        result <- div(textFieldComponent(addEvents), ul(children <-- listViews))\n      } yield result\n    }\n\n    OutWatch.renderInto(\"#app\",div(todoComponent)).unsafeRunSync()\n```
notice the `_  <- ` lines in the textFieldcomponent for comprehension....i'm not sure if this is clean or dirty, but it works ;)
@philbertwallace_twitter for your subscriptions `_ <- `, you could bind the subscription to a dom element, so they are atomatically canceled when the dom-element gets destroyed. Just search for `managed` in the changelog: https://github.com/OutWatch/outwatch/blob/changelog/CHANGELOG.md
@fdietze thanks for the additional pointers! One question regarding your comment about the `_ <- ...` lines, aren't they cleaned up by the for comprehension regardless? or are you saying I would have some possible memory leak of sorts with the way i currently have it?
As far as I know, they are not cleaned up automatically. The return a Monix `Cancelable` which would be discarded by `_ <- ...`
@philbertwallace_twitter yes, as felix said, the discarded subscription would not be cleaned up. The comprehension is about `IO`, not about observable subscription. But you can just move the subscriptions into your dom element, then they will be bound to lifetime of the dom element:\n```scala\n      result <- div(\n        input(onInput.value --> rawTextValues, onKeyPress.collect{case e if e.keyCode == KeyCode.Enter => e}.value.filter(_.length >= 4) --> candidate, value <-- clearEvents),\n        button(onClick(rawTextValues).filter(_.nonEmpty) --> candidate, \"Submit\", disabled <-- disabledFlag),\n        managed(outputEvents <-- candidate), //send output to parent component\n        managed(rawTextValues <-- clearEvents) //clear the input component\n      )\n```
I'm looking to collect the values of my input form and send them to the server.  Any suggestions on best practice here?  Here's as far as I've got (2 values only):    \n     \n         div(\n               input( onInput.value.map(id=>\"Id:\"+id) --> loginIdHandler),\n               input( onInput.value.map(xx=>\"FirstName:\"+xx) --> firstNameHandler),\n               input( onInput.value.map(xx=>\"Lastname:\"+xx) --> lastNameHandler),\n               button(\"Save\", onClick(loginIdHandler.combineLatest(firstNameHandler)) -->outputEventsHandler),\n               p(child<--outputEventsHandler.map(x=> x._1 +\",\"+x._2)\n             )\n\nI'd expect to create a Sink that Ajax-ed my data to the server (I'm using Autowire quite sucessfully).  
@marklister looks good so far! you can combine multiple handlers with factories from monix: for example `Observable.combineLatestN` or `Observable.combineLatestList`.  See https://monix.io/api/3.0/monix/reactive/Observable$.html#combineLatest3[A1,A2,A3](oa1:monix.reactive.Observable[A1],oa2:monix.reactive.Observable[A2],oa3:monix.reactive.Observable[A3]):monix.reactive.Observable[(A1,A2,A3)]. So you can use all of the handlers in your onClick:\n\n```\n           ...\n           button(\"Save\", onClick( Observable.combineLatest3(loginIdHandler, firstNameHandler, lastNameHandler) ) -→outputEventsHandler),\n```
outputEventsHandler would then need to be an `Observer[(String, String, String)]`

Thanks for the help.  I'm posting the following (working) code in case it helps anyone and also inviting comments on best practice...\n\n          object MemberEditPage {\n\n            val fields = Seq(\"Login Id\", \"First Name\", \"Last Name\")\n            val handlers: Seq[Handler[String]] = (1 to fields.length).map(x => Handler.create[String].unsafeRunSync())\n            val inp = fields.zip(handlers).map(x => inputComponent(x._1, x._2))\n            val merged = Observable.combineLatestList(handlers: _*)\n\n            def inputComponent(s: String, h: Handler[String]) = div(\n              label(s),\n              input(onInput.value.map(id => s + \":\" + id) --> h)\n            )\n\n            val saveButtonSink = Sink.create { x: Seq[String] => {\n              println(x)\n              IO(Ack.Continue)\n            }\n            }\n\n\n            val vnode = Future {\n              div(\n                div(\n                  cls := \"container-fluid\",\n                  div(cls := \"card card-header text-center\",\n                    h4(App.title)\n                  )\n                ),\n                div(cls := \"card card-header text-center\", h4(\"Edit Member\"),\n                  div(inp,\n                    button(\"Save\", onClick(merged) --> saveButtonSink),\n                    p(\"{\", child <-- merged.map(_.mkString(\",\")), \"}\")\n                  )\n                )\n              )\n            }\n          }\n
Next problem:  I have an `Observable[String]` containing the system time.  I provide this to an input as a default.  I can't see how to seed my Handler with the latest (or any) `Observable[String]`\n\nWhen I do this:\n```scala\ninput(id:=\"st\",tpe := \"time\", onInput.value --> startTimeHandler, value<-- Actions.systemTime)\n```\n\nThe handler is empty until I actually edit the number in the input.
`withLatestFrom`?

on the current master, we have refactored that part so a Handler ist just an alias for `Observer with Observable` instead of the current `Sink with Observer` where Sink was just a wrapper around Observer.



@marklister `Handler.create` and `Sink.create` are opinionated factories to enable you to write components in a safe way. Working with observables and their subscriptions as well as the dom itself has some sideffects. Using IO for the observable/observer and for the vdom-node assure it is safe to reuse this component in multiple places independently. 
if you are on master, `Handler` is just a type alias for `Observable with Observer`, so this should work on both backend and client?
It is fine, to not use `Handler.create` but just use another subject. The Handler factory is more of an opinionated, referentially transparent interface for creating subjects.


How is a `Sink` different from a `Handler`? I know that a Handler just uses a sink under the hood, but which should I use in what situation?
@Busti I think a `Handler` is both a `Source` and a `Sink`

Where a `Source` is analogous to an `Observable` and a Sink is similar to a `PublishSubject.limit(1)`, but with `.next` hidden?
A sink is like an `Observer`
But wrapped in an `IO`, right?
currently it is, but i am not sure whether it should. i think, the sideeffect comes from the `onNext`- function you are passing to the observer. so that should probably be wrapped in an IO and then you can map it into a Sink/Observable. it does not really help for referential transparency to put the observer itself in an IO. 
@cornerman That would also bring us back to my original plan of splitting the `ProHandler` away from the `Action -> State` pattern.


I have been thinking, I do not think that it would be smart to process effects and new actions sequentially like that.  \nIf you, for example, had an action that starts an animation, which creates a delayed effect, which stops the animation, the Store would basically halt while waiting for the effects Observable to complete.
And as far as the consistency of state is concerned, that depends on the perspective you are looking from.  \nThe state is still consistent, when looked at from the Subjects perspective. I.e. if you were to capture the state and a bunch of following actions at some point, you would always be able to reproduce the resulting state.
I am sorry, I cannot really phrase this any better.
@Busti ah true. I think you are right. Then we just keep the merge behaviour.
I also found the `scan0` operator in Monix, which behaves like scan, but it also emits it's initial element. That way we can remove the redundancy in `startWith`.


My state is mostly shared so i do it on the server.  I have used a `Handler` which is an `Observable` and `Observer` to keep a `Seq[String]` for me locally. it was pretty easy.  
`def res= resultHandler.scan(Seq.empty[String])((a,s)=>a:+s).map(_.map(div(_)))` 
That gives me every string that has been through `resultHandler`
How do you connect resultHandler with result of request to server?
`SecureClient[SecureApi].addTransaction(xx).call().map(r => resultHandler.onNext(r))`
it's autowire.  Returns a `Future[String]`
I think that's the same for diode.
I use this:\n```scala\neffectOnly(Effect(AjaxClient[Api].saveArticulo(item).call().map(UpdateArticulo)))\n```

where `UpdateArticulo`updates the data  state  in the diode circuit.

and is also autowire and return `Future[Articulo]`

Also with monix you can do `Observable.fromFuture(...)` which you can then wire up directly to things in the dom.  Makes the code quite neat.




@Busti , could you  tell me, how can i use createInputHandler, because 

@elyphas  The handler factory looks like this now: `handler.create[String].unsafeRunSync()`  To avoid gotchas always use a `Handler[String]`.  One can get rid of the `unsafeRunSync()` using `flatMap` and friends.  Have a look at the tests.  
`input(id := \"dur\", tpe := \"number\", onInput.value --> durationHandler, value <-- durationHandler)`
`Handler.create[T]`   sorry

@marklister @elyphas About the `Handler` factory: If you do not want your Handler in `IO`, you can do `Handler.unsafe[String]` instead of `Handler.create[String].unsafeRunSync`.
Now, by NOT using IO, you will be sharing the state of this Handler wherever you use it. For example when defining a component:\n\n```scala\nval counterComponent: VNode = {\n  val handler = Handler.unsafe[Int]\n  div(onClick(handler.map(_ + 1)) --> handler, \"Counter: \", handler)\n}\n```\n\nThen, if you use this component in multiple places (`div (counterComponent, counterComponent, counterComponent)`), then all this dom elements will share the same counter. With IO, you can make this state explicit:\n\n```scala\nval counterComponent: IO[VNode] = Handler.create[Int] { handler =>\n  div(onClick(handler.map(_ + 1)) --> handler, \"Counter: \", handler)\n}\n```\n\nIf you know reuse this component wrapped in IO, you will have three separate counters. This gives you a referentially transparent API, because you always get the same result. Otherwise, the state of the counterComponent depends on other elements and their user interaction. If you use IO and want to share the state, you can simply map over it: `counterComponent.map(c => div(c, c, c))` and you have the same shared state as before.


@marklister ; HI, sorry if I bother you with this question, but Could You help me please?, :)\nI don't know what is wrong with this?\n```scala\n    val resultHandler = Handler.create[String].unsafeRunSync().startWith(\"\")\n    for {\n      result <- AjaxClient[Api].getArticulo(\"029.101.0184\").call().map { r =>\n        val id = r.getOrElse(Articulo()).id\n        resultHandler.doOnNext(r => \"checar que actualize\" )  //map( r => id )\n      }\n    } yield ( println(\"No mames\") )\n\n    val root = div(\n      //resultHandler.scan(\"\")((s, id) => s ).map( r => div(\"checar ssssss\").map( r => r ) ),\n      div(textContent<-- resultHandler.map( r => r.toString ) ),\n      div(textContent:=\"Mostrar\")\n    )\n        OutWatch.renderInto(\"#root\", root).unsafeRunSync()\n```


```     \nval courts = Observable.fromFuture(SecureClient[SecureApi].courts().call())\n...\n\ndiv(\n        cls := \" row justify-md-content-center \",\n        Actions.courts.map(_.map(x => courtHtml(x,user))\n        )\n      )\n```

And the Handler factory's startWith method:\n\n```\n    val dateHandler = Handler.create[String](todayString).unsafeRunSync()\n    val userHandler = Handler.create[String](\"\").unsafeRunSync()\n    val amountHandler = Handler.create[String](\"0.00\").unsafeRunSync()\n```

@elyphas  here I'm sending an update to the server and getting the result back:\n```\n    val resultHandler= Handler.create[String](\"\").unsafeRunSync()\n    def res= resultHandler.scan(Seq.empty[String])((a,s)=>a:+s).map(_.map(div(_)))\n\n    def saveButtonSink = Sink.create {\n\n      x: Seq[Any] => {\n        println(x)\n        val t1 = Try{Transaction.from(0+:x)}\n        t1.map { xx =>\n          val t = xx.copy(description1 = \"Payment thank you: \" + xx.description1)\n            .copy(description2 = \"Received from debtor: \" + xx.account1 + \" \" + xx.description1)\n            .copy(account1 = xx.account1 + 1200)\n          SecureClient[SecureApi].addTransaction(xx).call().map(r => resultHandler.onNext(r))\n          amountHandler.onNext(\"0.00\")\n          userHandler.onNext(\"\")\n          referenceHandler.onNext(\"\")\n\n          }\n          Continue\n\n      }\n    }\n```

@marklister ; thank you again, although, I need to use `unsafeOnNext` instead `onNext` so it is like this:\nbut it helps me a lot, \n```scala\n    val resultHandler= Handler.create[String](\"\").unsafeRunSync()\n    def res: Observable[Seq[VNode]] = resultHandler.scan(Seq.empty[String])((a, s) => a :+ s ).map( _.map( div( _ ) ) )\n\n    for {\n      result <- AjaxClient[Api].getArticulo(\"029.101.0184\").call().map { r =>\n        val descri = r.getOrElse(Articulo()).descripcion.getOrElse(\"\")\n        resultHandler.unsafeOnNext(descri)\n      }\n    } yield ( println(\"No mames\") )\n\n    val root = div(\n      children <-- res,\n    )\n\n    OutWatch.renderInto(\"#root\", root).unsafeRunSync()\n```
Now this way; :) \n```scala\n    val count = Observable.fromFuture(AjaxClient[Api].getArticulo(\"029.101.0184\").call())\n\n    val root = div(\n      //children <-- res,\n      child <-- count.map { v: Option[Articulo] =>\n        val ver = v.getOrElse( Articulo() )\n        ver.descripcion.getOrElse(\"\") + \"Changos -+-+-+-+-\"\n      } ,\n    )\n    OutWatch.renderInto(\"#root\", root).unsafeRunSync()\n```


nice! Be aware when using an `Observable.fromFuture`, the future might fail and then the Observable would call `onError` on its observers. Now, if outwatch sees an error in any Observable of a VNode, it will stop patching this VNode. This strict error handling might or might not be desired, but currently that is how it is implemented. the whole VNode where a failed Observable is used will become unreactive. So I would recommend to do `Observable.fromFuture(...).onErrorRecoverWith { case t => log(t); Observable.empty }` when working with API calls. Wrapping the future in a task like `Task.deferFuture(...)` can help for building in a retry logic.
There is a way to at least get notified when something like this happens in your app: `OutwatchTracing.error.foreach { t => showErrorPage }`. OutWatch will additionally log an error to the console.
@cornerman yes, thanks!  I saw `OutwatchTracing.error` recently...  It will be good to see someone publish an outwatch app that has all the best practices down pat.  As you can see I got started before everthing got wrapped in `IO` and transitioned to `.unsafeRunSync()` leaving some very ugly code.




I hope that the IO situation became a bit better with the current master, it should be possible to get around most IO stuff (except Outwatch.renderInto), when you stick to `Handler.unsafe`.
Hi, someone could tell me what would be the best way to combine differentes sources I use this for two:\n```scala\n    val txtRFC = Handler.create[String](\"\").unsafeRunSync()\n    val txtRasonSocial = Handler.create[String](\"\").unsafeRunSync()\n    val txtRepresentante = Handler.create[String](\"\").unsafeRunSync()\n\n    val datosProveedor = txtRFC.combineLatestMap(txtRasonSocial)( (f, s) => f + s )\n\n    val root = div(\n      label(\"R.F.C.\"),\n      input(onInput.value --> txtRFC ),\n      label(\"Razon Social\"),\n      input(onInput.value --> txtRasonSocial ),\n      label(\"Representante\"),\n      input(onInput.value --> txtRepresentante ),\n      button( \"Clickear\", onClick(datosProveedor) --> saveButtonSink )\n    )\n    OutWatch.renderInto(\"#root\", root).unsafeRunSync()\n```
by the way, I want to fill a `case class`:\n```scala\ncase class Proveedor(\nid: String = \"\", descripcion: Option[String] = None,  propietario: Option[String] = None, calle: Option[String] = None, colonia: Option[String] = None,  delegacion: Option[String] = None, cp: Option[String] = None,  ciudad: Option[String] = None, telefonos: Option[String] = None, fax: Option[String] = None, observaciones: Option[String] = None, activo: Boolean = true,  elaboro: Option[String] = None,  giro: Option[String] = None, descuento:  Option[String] = None )  \n```\nso there are many sources, :)
@elyphas You could use the cats typeclass `Parallel` for this, as monix provides `NonEmptyParallel` instances for Observable. What this means in practice:\n\n```scala\n      import cats.implicits._\n      (observable1, observable2, observable3).parMapN { (a,b,c) => a.toString + b.toString }\n```
@elyphas additional feedback to your previous example with `Sink.create`. You construction of the IO should look a bit different, when you want to model the side effect:\n\n```scala\n  def saveButtonSink = Sink.create[String] { x: String =>\n      IO {\n        println(x)\n        Ack.Continue\n     }\n  }\n```

@cornerman ; Hi, Could You help me please?, :)\nI have an error in this block of code:\n```scala\n    def saveButtonSink = Sink.create[String] { x: String =>\n      IO {\n        println( x )\n        Continue\n      }\n    }\n```\nI have this error:\n`Expression of type IO[Ack.Continue.type] doesn't conform to expected type Future[Ack]`
and this code:\n```scala\n(txtRFC, txtRasonSocial, txtRFC).parMapN { (a, b, c) => a.toString + b.toString + c.toString }\n```\nI have an error:\n```scala\nNo implicits found for parameter p: NonEmptyParallel[Handler, F_]\n```
@elyphas ah sorry, i thought you were using an older version. on master, you just leave out the `IO` in the sink. Or use `onClick.foreach { x => .... }`.
for the implicits: do you have an `import cats.implicits._`?






@elyphas does a simple example compile: `  val x: Observable[String] =(Observable.empty[Int], Observable.empty[String]).parMapN { (a,b) => a.toString + b.toString}` ?
also please try to provide the type args explicitly: `.parMap[CombineObservable.Type, String] { (a, b, c) => a.toString + b.toString + c.toString }`














I think, you could simplify your code a lot if you would have a data class to represent your txt handler `case class Txt(rfx: String, description: String, ...)`. 

for your components later you would one handler and then map it to the right part of the case class: `handler.map(_.rfc)`, `handler.map(_.description)`, and so on


I am thinking that I need a `var lstProveedor : Seq[Proveedor]` to query in the future.
then just keep that and create one `Handler[Proveedor]`. you do not really need to have a single handler for each part of it. you can create a sink that will zoom into the case class, for example:\n\nsomething like this should work and we should have a helper for that:\n```scala\ncase class Data(number: Int, text: String)\nval handler = PublishSubject[Data]\n\ndiv(\n  onInput.value.transform(_.withLatestFrom(handler)((txt, data) => data.copy(txt = data))) --> handler\n)\n\n```
by using `withLatestFrom` you are essential replacing your `var lstProveedor` (to keep the current state) by an observable operation that combines the latest value when emitting a new onClick event

onClick was a bad example, it would need an event that emits a string, like `onInput.value.transform....` or if you have a constant, you can simply do: `onClick(handler).map(_.copy(txt = \"newTxt\")) --> handler`

@cornerman; sorry,\nI got troubles with a component :\n```scala\n    val handlerProveedor = PublishSubject[Proveedor]\n\n    val txtRFC = handlerProveedor.map( _.id ) //Handler.create[String](\"\").unsafeRunSync()\n\n    //def component( lbl: String, h: ProHandler[ String, String ] ): VNode = {\n    def component( lbl: String, h: PublishSubject[String] ): VNode = {\n      div( //width:=\"500px\",\n        label( lbl, backgroundColor:=\"gray\", fontWeight := 10, marginRight:=\"10px\" ),\n        input( inputString --> h, value <-- h, marginRight := \"20\" )\n      )\n    }\ncomponent( \"R.F.C.\", txtRFC ),\n```\n```scala\nType mismatch, expected: PublishSubject[String], actual: Observable[String]\n```
Yeah, fair enough. The map only creates an observable. We have the method `.lens` for handlers and subjects (https://github.com/OutWatch/outwatch/blob/master/outwatch/src/main/scala/outwatch/MonixOps.scala#L72). That allows to map the observable as well as observer part of a handler. But it is not that easy to use, because for the observer to works, it needs to be subscribed. So you could do:\n\n```scala\nval txtRFC: Handler[String] = handlerProveedor.lens[String](Proveedor.initial)( _.id )((proveedor, id) => proveedor.copy(id = id))\nval cancelable = txtRFC.connect() // need to subscribe to this handler, because it internally needs to track the current state.\n```

@cornerman ; Hi, Would You give me your point of view about this code?\nThis works nicely, it do what I want, :)\n```scala\n    val onClickItem = Sink.create[Option[Proveedor]] { p: Option[Proveedor] =>\n      p.map { i =>\n        txtRFC.onNext(i.id)\n        txtRepresentante.onNext(i.propietario.getOrElse(\"\"))\n        store.onNext(Clean) //Clean the store to start again.\n      }\n      Continue\n    }\n    def gridComponent(s: AppState) = div ( position:=\"absolute\",  zIndex:=1000, backgroundColor := \"#f99d89\",\n        s.lst.map { i =>\n        div ( i.id + \": \" + i.descripcion.getOrElse(\"\"), width := \"500px\", border:=\"1px solid\",\n          onClick( s.lst.filter( p => p.id == i.id).headOption   ) --> onClickItem\n        )\n      }\n    )\n    def component( lbl: String, h: ProHandler[ String, String ] ): VNode = {\n      div( width:=\"800px\",\n        label( lbl, backgroundColor:=\"gray\", fontWeight := 10),\n        input( onInput.value.map(r => r.target.value.toString) --> h, value <-- h, marginRight := \"20\" )\n      )\n    }\n    val root = div ( width:=\"20px\", position := \"relative\",\n      component( \"R.F.C.\", txtRFC ),\n      div( width:=\"800px\", margin := \"10px\",\n        label( \"Razon Social\", backgroundColor:=\"gray\", fontWeight := 10 ),\n        input(  onInput.target.value --> txtDescripcion,\n                onInput.target.value --> onSearchProveedor,\n                value <-- txtDescripcion\n        ),\n      ),\n      store.map { s => gridComponent( s ) },\n      component( \"Representante\", txtRepresentante ),\n    )\n    OutWatch.renderInto(\"#root\", root ).unsafeRunSync( )\n```






@elyphas It is a bit rough around the edges.  \nI guess for now only having 2 files is fine, but you would definitely have some scalability problems once you try to do something bigger.  \n* The dom types support currying. That means that you can write:  \n  ```scala\n    div(cls := \"foo\")(\n      p(\"Cool content in a paragraph\")\n  )\n  ```\n  Which makes your code look a bit more *html-like*\n* You seem to have a bunch of styling values mixed into your code.  \n  Generally it would probably be better to move those to a **css** file and only add `class` attributes to your html tags.  \n  I also do not really like the styling tags everywhere. I know that using `div(width := \"42px\")` or `div(backgroundColor := \"tomato\")` translates to css, but if I really had to dynamically style something like that I would probably use `div(style.background-color := \"tomato\")`. But maybe @cornerman also has some other thoughts on that. But since that is defined by `scala-dom-tags`, this is probably the wrong place to discuss that.  \n  The general idea here is that html is for data and markup info and the **.css** file provides the styling and layouting.  \n* There probably are a few places where you could define your component functions inside of the scope of another function to avoid passing an unnecessary amount of parameters.  \n  ```scala\n  def application(state: Observable[MyApplicationModel]) = {\n    def someComponent(foo: String) = h2(foo, state.map(_.foo))\n    \n    div( cls := wrapper)(\n      h1(\"Welcome, \", state.map(_.user.name)),\n      someComponent(\"Hello\"),\n      someComponent(\"World\")\n    )\n  }\n  ```



@Busti ; Hi, Could You tell me, please, :) ,  why the return type of Store changed to this:\n```scala\nIO[ProHandler[A, (A, M)]] = IO {\n```\nI needed to change this:\n```scala\n  def gridEditable(state: Observable[(ActionViewRenglonRequisicion, NukulState)],\n                   store: Observer[ActionViewRenglonRequisicion]): VDomModifier\n```\nand this code with `._2` to acces to the state:\n```scala\ncurrent._2.renglonActive\n```\nis it  right what I did?
@elyphas Store now returns an `Action -> State` tuple. This is really useful when you want information about the event that lead to the current state.  \nI personally have three uses for this.  \n 1. I use it to implement an **undo** **redo** (ctrl + z / ctrl + shift + z) functionality, in which I save the actions that have gone into a store and their state and then unapply the action / roll back to the previous state.  \n 2. I use it to sync 2 components on different clients via network by sending just the actions between the two clients. This saves a ton of bandwidth and has the added bonus that I can debounce based on the actions.  \n 3. It allows you to update certain parts of your view only when a specific action is fired.\n\nYou can always do `map(_._2)` to get rid of the action, but the more idiomatic way would probably to use pattern matching in the `map` function like this: `store.map { case (action, state) => state }`.  \nYou can also use this to filter certain actions: `store.collect { case (SomeAction, state) => state.varMutatedByAction }`




Hi, someone could help me? please;\nI tried to set a value by default:\n\n```scala\n    val datosGralesRequi = DatosGralesRequisicion( cve_oficina = \"1221\" )\n    val handlerDatosGrales = PublishSubject[DatosGralesRequisicion]()\n   //this way \n    ///handlerDatosGrales.behavior( datosGralesRequi )\n    // and this way\n    //handlerDatosGrales.onNext(datosGralesRequi)\ninput(\n            value <-- txtCveOficina.map( r => r ),\n            onChange.target.value --> txtCveOficina,\n            width := \"60px\"\n ),\n```\nbut I can't do it.\nalso this way:\n```scala\nval txtCveOficina = handlerDatosGrales.lens[String]( datosGralesRequi )( _.cve_oficina )((datosGrales, cve_oficina) => datosGrales.copy(cve_oficina = cve_oficina))\ntxtCveOficina.onNext(\"1221\")\n```
@elyphas Have you tried using `.startWith`?
@elyphas as @Busti said, you need your observable to always start with a \n certain value (`datesGralesRequi`). Then `startWith` is what you want. If you want to just always emit the latest value as the first one, you could use a `BehaviourSubject[DateGralesRequisicion](datosGralesRequi)`.
@elyphas In that case you can also use `Handler.create(datosGralesRequi)` which is a utility for creating a `BehaviorSubject` wrapped in an `IO`.\nI usually use for comprehensions to create these:\n```scala\nval foo = for {\n  datosGralesRequi = DatosGralesRequisicion( cve_oficina = \"1221\" )\n  handlerDatosGrales <- Handler.create(datosGralesRequi)\n} yield input(width := \"60px\")(\n  value <-- txtCveOficina,\n  onChange.target.value --> txtCveOficina\n)\n```  \n\n`Handler.create[Option[String]](None)` also accepts a type parameter, in the case that your type cannot be inferred, or you want the type to be lower than what you initialize the value with.



I have another problem:\n```scala\n    val datosGralesRequi = DatosGralesRequisicion (cve_oficina=\"1221\", fecha=Fechas(\"01/01/2019\"), ejercicio=2019, ejercicio_presupuestal=2019 )\n    val handlerDatosGrales = Handler.create[DatosGralesRequisicion](datosGralesRequi).unsafeRunSync() //PublishSubject[DatosGralesRequisicion]()\n    handlerDatosGrales.onNext ( datosGralesRequi )\n\n    val txtIDFuente = handlerDatosGrales.lens[String]( datosGralesRequi )( _.fuente_financiamiento )((datosGrales, fuente_financiamiento) => datosGrales.copy(fuente_financiamiento = fuente_financiamiento))\n    val cancelableIDFuente = txtIDFuente.connect()\n\n    val txtFuente = handlerDatosGrales.lens[String]( datosGralesRequi )( _.fuente_financiamiento )((datosGrales, fuente_financiamiento) => datosGrales.copy(fuente_financiamiento = fuente_financiamiento))\n    val cancelableFuente = txtFuente.connect()\n```\nI tried this to update one field \n```scala\n    val onClickItem = Sink.create[ String ]{ f =>\n      //txtIDFuente.onNext(f) //this way doesn't works\n      for {\n        c <- handlerDatosGrales\n      } yield {\n              handlerDatosGrales.onNext ( c.copy(fuente_financiamiento = f) )\n      }\n      Continue\n    }\n```\nbut doesn't works\ncould Yout tell me what is the right way?
`handlerDatosGrales` is an io. when you do for in the `onClickItem` it is not actually executed, just mapped. So you will need to map over the `handlerDatosGrales: IO[Handler]` outside of the onClickItem.
if you want to just share this state, you could just do `Handler.unsafe`, and use it where you need it. If you want to be rt, you want to map over the IO, build your component and then render the IO.

Hi, someone could help me?, please,\nI want to generalize this function:\n```scala\ndef cmpInput( lbl: String, hdl: Handler[String], w: Int ) = {\n      div( width := w.toString + \"px\", label(lbl),\n        input (\n          value <-- hdl.map( r => r ),\n          onChange.target.value --> hdl,\n          width := \"60px\"\n        )\n      )\n    }\n    def cmpInputInt( lbl: String, hdl: Handler[Int], w: Int ) = {\n      div( width := w.toString + \"px\", label(lbl),\n        input (\n          value <-- hdl.map( r => r ),\n          onChange.target.value.map(r=> r.toInt) --> hdl,\n          width := \"60px\"\n        )\n      )\n    }\n```\nI tried this: \n```scala\ndef cmpInput[T]( lbl: String, hdl: Handler[T], w: Int ) = {\n      div( width := w.toString + \"px\", label(lbl),\n        input (\n          value <-- hdl.map( r => r ),\n          onChange.target.value --> hdl,\n          width := \"60px\"\n        )\n      )\n    }\n```\nBut has errors:
@elyphas The `hdl` is of type `Observer[T]` and not of type `Observer[String]`. Because `onChange.target.value` will emit the current value of the `event.currentTarget` element as a string. The same is true for `value <-- hdl.map( r => r )`, where you need an `Observable[String]`. From what I see, you do not need to make `cmdInput` generic, just require a `Handler[String]`.
you could redirect your handler when calling the method with other type and parse the value to and from string. For example for the int case:\n\n```scala\nval handler: Handler[Int] = ???\ncmdInput(\"Some Label\", handler.mapHandler[String](_.toInt)(_.toString), 100)\n```\n\nor have have some error handling for the int<=>string conversion (i should add that method to outwatch):\n\n```scala\nimplicit class HandlerWithMaybe[T](val self: Handler[T]) extends AnyVal {\n  def mapHandlerMaybe[T2](write: T2 => Option[T])(read: T => T2): Handler[T2] = outwatch.ProHandler(self.redirectMapMaybe(write), self.map(read))\n}\n\nval handler: Handler[Int] = ???\nval strHandler: Handler[String] = handler.mapHandlerMaybe[String](s => Try(s.toInt).toOption)(_.toString)\n```
@cornerman ; sorry if I bother You with this question, but Could You tell me if this code is something related to `imap`?\n```scala\nhandler.mapHandlerMaybe[String](s => Try(s.toInt).toOption)(_.toString)\n```
@elyphas about `mapHandlerMaybe` and `imap`: `imap` is provided by cats with monix defining the needed instances. It is very similar but mind the signature: `def imap[R](f: T => R)(g: R => T): Handler[R]` and `def mapHandlerMaybe[R](f: R => Option[T])(g: T => R): Handler[R]`. The first one defines a one-to-one mapping between the two types. The latter defines a handler of type `R`, but not every value of type `R` can be transformed into the target type. So we return an option and drop every value that is None.
Hi, someone could tell me? please, if this is a right way to condition?\n```scala\n      Observable.fromFuture {\n        for {\n          a <- AjaxClient[Api].insertRenglonRequisicion(renglon).call()//.map { a => a.getOrElse( ViewRenglonRequisicion()) }\n        } yield a match {\n          case Left(v) =>\n            org.scalajs.dom.window.alert(v)\n            Observable.empty\n          case Right(v) => Observable(v)\n        }\n      }.flatten\n```


@elyphas It is a possibility and depends entirely on how you want to handle errors. I would probably write it with map instead of using a for comprehension and then match, but it is fine.
Be aware that your AjaxClient future might fail and if you render an observable with an error, outwatch will stop updating this VNode. So you might want to `transform` your future  to handle errors as well or `handleErrorWith` on the observable.
So, you could do something like this:\n\n```scala\nObservable.fromFuture(AjaxClient[Api].insertRenglonRequisicion(renglon).call())\n  .onErrorRecover { case NonFatal(t) => Left(s\"Error: ${t.getMessage}\") }\n  .flatMap {\n          case Left(v) =>\n            org.scalajs.dom.window.alert(v)\n            Observable.empty\n          case Right(v) => Observable(v)\n }\n```

@elyphas thank you for your second example, is it typical to create handlers using `for` as you do here\n\n```\nfor{\n  hdl <- Handler.create[String]\n}...\n```\n\nand return a IO[...]?  I have been creating them like this\n\n```\n val titleHandler: Handler[String] = Handler.create[String](document.title).unsafeRunSync()\n```\n\nAlso I'm not sure you need to include the server and shared code for this example.
@babyman_twitter We typically create Handlers in `IO`, because you are typically creating a stateful element.\n\nTake for example:\n```scala\nval myComponent = {\n  val titleHandler: Handler[String] = Handler.create[String](document.title).unsafeRunSync()\n  div(titleHandler, onClick(\"New Title\") --> titleHandler)\n}\n```\n\nNow, if you would use `myComponent` multiple times in the dom, you would share the state for all these elements. Example:\n```scala\ndiv(myComponent, \"Something else\", myComponent)\n```\n\nIf you would click either of the two myComponent, the titleHandler would be set to \"New Title\" in both components that you have rendered, because both use the same handler.\n\nIf you build them with IO instead of doing `unsafeRunSync`, you have two different handler for the two components, because the IO for the components will be run for each of the two myComponents you are rendering. Example:\n```scala\nval myComponent = Handler.create[String](document.title).map { titleHandler =>\n  div(titleHandler, onClick(\"New Title\") --> titleHandler)\n}\n```\n\nYou would now have two different/unreleated myComponents if you use them twice:\n```scala\ndiv(myComponent, \"Something else\", myComponent)\n```\n\nOr you do share the state explicitly by mapping over the io:\n```scala\nmyComponent.map { myComponent =>\n  div(myComponent, \"Something else\", myComponent)\n}\n```\n\nIt allows you to define referential transparent functions for your components.
As a sidenote, there is a shortcut for `Handler.create[T](...).unsafeRunSync`, that is: `Handler.unsafe[T](...)`.
If you do not care about sharing state, you can just do `Handler.unsafe` and ignore the whole IO stuff. If you do care, it can help you when reusing components. 
@cornerman that makes sense because the `IO` defers execution(?), and since my components are defined as functions rather than as vals I have not seen the issue.  It seems rewriting my component below to return an IO[BasicVNode] would be more in the spirit of OutWatch if nothing else.  Otherwise does this code seem reasonable?\n\n```\n  private def tagAddBox(addTagSink: Observer[String]): BasicVNode = {\n\n    val tagText = Handler.create[String](\"\").unsafeRunSync()\n\n    val disableTagAdd = tagText.map(_.length < 3)\n\n    div(cls := \"field is-grouped\",\n      p(cls := \"control is-expanded\",\n        input(`type` := \"text\",\n          cls := \"input\",\n          value <-- tagText,\n          onInput.value --> tagText,\n          placeholder := \"tag name\")),\n      p(cls := \"control\",\n        button(`type` := \"button\",\n          cls := \"button\",\n          disabled <-- disableTagAdd,\n          onClick(tagText) --> addTagSink,\n          onClick(\"\") --> tagText,\n          \"add tag\")))\n  }\n```
Ahhhh, wait, am I right to think that by using a function to define the component it captures state (in the closure) so as not to be internally pure even though it looks like pure FP code to the caller?  The `IO` return allows the internals to remain pure also?  I have a lot to learn...
@babyman_twitter exactly. The `IO.apply` defers the execution and is run when it is rendered. `IO.pure` would be the strict constructor. \n\nThe code looks good to me. Using IO would maybe be more outwatch-like, but both is fine.\n\nHere a small explaination for referential transparency: https://softwareengineering.stackexchange.com/a/340904\n\nTaking this definition of RT for your example `tagAddBox(observer)`. If I had `div(tagAddBox(observer), tagAddBox(observer))`, then I could not replace all occurrences of `tagAddBox`-function-call with the result. This is not the same as `val box = tagAddBox(observer); div(box, box)`. With IO, it would be the same, because then the function just returns how to compute the value and not the value itself.\n\n
