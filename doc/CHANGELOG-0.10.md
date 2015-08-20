# 0.10.0 (unreleased)

* Replaced Router v1 with v2.

  `extra.router` has been removed; `extra.router2` has been renamed to take its place.

  If for some reason, you want to use Router v1, don't want to migrate to v2 *and* want to keep up-to-date with scalajs-react,
  please copy Router v1 into your own codebase.
  Maintaining two Routers in scalajs-react is not good.

* [#145](https://github.com/japgolly/scalajs-react/issues/145): Unify `-->` `==>` `~~>`, remove Scalaz `IO`, add `Callback`.

  **Problem:**<br>
  A gap in the community and library was growing because there were two separate ways of writing callbacks.
  An example of which, numerous users using the built-in `Router` experienced confusion and were forced to
  discover Scalaz's (and Haskell's) `unsafePerformIO()` without understanding the reasons or benefit.
  Another example, writing mixins or shared libraries, one would have to create duplicate methods with differing
  type signatures in order to accomodate both styles.
  More examples exist, but it should be clear that this please-everyone approach has failed.

  **Solution:**<br>
  A new data type `Callback` has been added
  (which is actually an alias to the real type `CallbackTo[A]` where `Callback = CallbackTo[Unit]`).
  It replaces the existing two competing methods of `() => Unit` and Scalaz `IO[Unit]`.
  A basic rule of thumb is if a function performs an effect (changing DOM, using network, changing React state),
  it should be wrapped in a `Callback`.

  * When creating HTML, `-->` and `==>` only accept `Callback`s.
  * `setState()`, `modState()` etc now return `Callback`s.
  * Useful methods now moved into core `-->?`, `==>?` which work with optional callbacks.
  * Useful methods now moved into core `_setState()` and `_modState()`.
  * Component lifecycle methods (like `componentWillMount`) now accept `Callback`.
  * `Router` no longer uses Scalaz `IO`.
  * Mixins in `extra` no longer have duplicate methods for Scalaz `IO`.

  For Scalaz users:<br>
  Either use `CallbackTo` instead of `IO`, or else change your `IO`s to `CallbackTo`s when you pass them to React.
  * `~~>` is removed. Use `-->` and `==>` instead.
  * `{,_}{set,mod}StateIO` methods all removed and in core without the `IO`.
  * Isomorphism between `IO` and `CallbackTo` with convenience extension methods `.toCallback` and `.toIO`.
  * `OpCallbackIO` has been removed in favour of plain `Callback`.
  * New state-monad type-inference convenience: `FixCB[S] = FixT[CallbackTo, S]`

  **Example migrations**<br>

  Example #1
  ```scala
  // Before
  val Example = ReactComponentB[Unit]("Example")
    .initialState(0)
    .render { $ =>

      def clickHandler = $.modState(_ + 1)
      <.div(
        <.div("Button pressed ", $.state, " times."),
        <.button("CLICK ME", ^.onClick --> clickHandler))

    }.buildU

  // After
  // Surprise! No change needed.
  // modState() now returns a Callback which is exactly what --> expects.
  ```

  Example #2<br>
  ```scala
  // Before
  def clickHandler: Unit = {
    println("Updating state.")
    $.modState(_ + 1)
  }

  // Callbacks can compose in many different ways.
  // The following examples all do the same thing in the same order.

  // Method #1: >>
  def clickHandler: Callback =
    Callback(println("Updating state.")) >>
    $.modState(_ + 1)

  // Method #2: precedeWith
  def clickHandler: Callback =
    $.modState(_ + 1).precedeWith {
      println("Updating state.")
    }

  // After, Method #3: <<
  def clickHandler: Callback =
    $.modState(_ + 1) << Callback(println("Updating state."))

  // After, Method #4: runNow()
  def clickHandler = Callback {
    println("Updating state.")
    $.modState(_ + 1).runNow()
  }
  ```

  Example #3: Snippet from https://japgolly.github.io/scalajs-react/#examples/todo<br>
  ```scala
  // Before
  def handleSubmit(e: ReactEventI) = {
    e.preventDefault()
    $.modState(s => State(s.items :+ s.text, ""))
  }

  // After
  def handleSubmit(e: ReactEventI) =
    e.preventDefaultCB >>
    $.modState(s => State(s.items :+ s.text, ""))
  ```

* The `extra` module no longer depends on Scalaz or Monocle.

  `extra` only depends on `core` now.
  The built-in Router now relies only on scalajs-react (`Callback` was very important for this), and Scala stdlib.
  
  Methods previously in `extra` that use Scalaz or Monocle (for example, `Reusability.byEqual`, `ReusableVar#setL`)
  now require you to import `ScalazReact._` and/or `MonocleReact._`.
  After doing so, the methods will appear to be available as if nothing has changed.

<br>
Migration commands:
```sh
# extra.{router2 ⇒ router}
find . -name '*.scala' -type f -exec perl -pi -e 's/(?<=extra\.router)2//g' {} +

# Unfortunately the migration to Callback is mostly manual.
# Here are some commands I used to help but they'll only get you halfway there.
find . -name '*.scala' -type f -exec perl -pi -e 's/(?<=forceUpdate)\(\)//g' {} +
find . -name '*.scala' -type f -exec perl -pi -e 's/(?<=tryFocus)\(\)//g' {} +
find . -name '*.scala' -type f -exec perl -pi -e 's/(?<=(set|mod)State)IO//g' {} +
find . -name '*.scala' -type f -exec perl -pi -e 's/FixT\[IO *, */FixCB[/' {} +
find . -name '*.scala' -type f -exec perl -pi -e 's/ReactST\[IO *,/ReactST[CallbackTo,/' {} +
```