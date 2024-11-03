# Collecting User Input

Clicking a button is one way for users to interact with your application. Another way of interaction is writing inside input boxes. Often the user must fill in the information in a form, such as the username and password on a login page.

### Textual Inputs

The simplest input field is a text box to collect raw textual information. Imagine that we are building the following sample program:

<resolved-image source="/images/elm/text-input.gif" />

It consists of just two elements: a text input box and a label. The text value of the label is whatever the user types into the text input box. This means that the only data that the application tracks is a single string. We can start modeling the `State` and `Msg` types as follows:

```fsharp
type State = { TextInput : string }

type Msg =
    | SetTextInput of string
```
Notice that to handle the input of a single field in an Elmish application, we need a field in a `State` type: the `TextInput` field. We also need an event type `Msg` with a case `SetTextInput` that will be triggered when the text of the input changes. When the `SetTextInput` is triggered, we *manually* set the `State`'s `TextInput` value inside the update function. This is the big difference between The Elm Architecture and other UI architectures such as MVVM which include "two-way" binding features. In The Elmish Architecture, there is only "one-way" binding and the updates are coded explicitly.

> In Elmish applications, it is common to name the input fields `{FieldName}` for the `State` and `Set{FieldName}` for the event. `On{FieldName}Changed` is also another option for naming the event associated with the field.

The implementation of `init` and `update` are self-explanatory:
```fsharp
let init() = { TextInput = "" }

let update msg state =
  match msg with
  | SetTextInput textInput ->
      { state with TextInput = textInput }
```
What's more interesting is the `render` function:
```fsharp {highlight: [3]}
let render state dispatch =
  Html.div [
    Html.input [ prop.onChange (SetTextInput >> dispatch) ]
    Html.span state.TextInput
  ]
```
Notice here the `onChange` event handler takes a function of type `string -> unit` where the input is the current value of the input element. This function is triggered every time the value changes, which in turn dispatches the `SetTextInput` message into our Elmish program and into the update function.

When the state is updated with the new `TextInput`, the user interface is re-rendered and thus the `span` (highlighted below) will reflect the current value of the text input element:
```fsharp {highlight: [4]}
let render state dispatch =
  Html.div [
    Html.input [ prop.onChange (SetTextInput >> dispatch) ]
    Html.span state.TextInput
  ]
```

We should mention here that even when the UI is re-rendered, the input box element preserves the text that has been entered. We *didn't* tell the `input` element that it should use `state.TextInput` but still with each render cycle, the text is kept as it is. This makes sense because if it *did* reset the internal state with every cycle, then the text box would be cleared after each keyboard stroke which is not exactly what your users would be expecting.

This happens because HTML input elements such as a text box keep track of an *internal* state of their own so we don't have to tell the input box what text it should have now. However, sometimes we do want to tell the input box which value it should have, for example when initializing the application, maybe we don't want the input element to start empty but instead have an initial value. For this, we would use the `valueOrDefault` property that forces the input element to change the internal state:

```fsharp {highlight: [4]}
let render state dispatch =
  Html.div [
    Html.input [
      prop.valueOrDefault state.TextInput
      prop.onChange (SetTextInput >> dispatch)
    ]

    Html.span state.TextInput
  ]
```
Now if we had the initial state set to `{ TextInput = "Some initial text" }` then the input box would start with that text instead of starting from an empty string. So far so good. There is a lot more to input elements but for now we know how to handle the basic case of text elements. It's now time to handle numeric inputs.

### Numeric Inputs: Validation and Transformation

At first glance, handling numeric inputs isn't that much different than handling textual input except for the fact that we keep track of a number (float or integer) instead of a string. But it likely does add an extra step to the problem: handling validation and transforming values.

Let's start with `State` and `Msg`:
```fsharp
type State = { NumberInput : int }

type Msg =
    | SetNumberInput of int
```
Just like our previous example, this one also has a field `NumberInput` on the state type and the event `SetNumberInput` on the messages type. Again, the `init` and `update` are self-explanatory:
```fsharp
let init() = { NumberInput = 0 }

let update msg state =
  match msg with
  | SetNumberInput numberInput ->
      { state with NumberInput = numberInput }
```
Now in the `render` function, how can we trigger `SetNumberInput` from the `Html.input` element if the `onChange` event takes a string? That is simple. Just parse as an integer before dispatching the `SetNumberInput` event:
```fsharp
prop.onChange (int >> SetNumberInput >> dispatch)
```
And the `render` function will look like this:
```fsharp {highlight: [5]}
let render state dispatch =
  Html.div [
    Html.input [
      prop.valueOrDefault state.NumberInput
      prop.onChange (int >> SetNumberInput >> dispatch)
    ]

    Html.span state.NumberInput
  ]
```
Now as you read the code snippet above, I am hoping that all kinds of alerts are going off in your head because this code is very, very problematic! Can you spot the bug?

The specified event handler above will try to parse *every* input into an integer and will fail for inputs that are not properly formatted as integers, throwing an exception in the process!

<resolved-image source="/images/elm/int-exn.gif" />

Oh my, an exception in an Elmish application?! This is definitely a worst-case scenario because it broke a bunch of stuff. The event handler errored which means the `update` function isn't called. That caused the UI to go out-of-sync with the data. And all of this is because we tried to parse the input as an `int` in an unsafe manner without checking whether the parsing was actually successful or not.

In functional programming, the `int` function that parses the input is called a "partial" function. `int` is a partial function because it can only successfully evalute a subset of all possible inputs. That is to say, there are only some strings that the `int` function can successfully convert to the output type (an integer). For some input strings (the ones that are not properly formatted as integers) the function will throw. Partial functions are considered unsafe exactly because they throw and cause unexpected behaviour in the application.

To remedy partial functions, we can make them safe by turning them into "total" functions. A total function does not throw and can handle every input to produce an output whether it was successful or not. The output of such a function usually returns an `Option<'t>` or a `Result<'t, 'err>` value but other detailed output types are possible as well. Here is a "total" or "safe" version of the `int` function:
```fsharp
let tryParseInt (input: string) : Option<int> =
    try Some (int input)
    with | _ -> None
```
Notice the different type signatures of the two functions:
```fsharp
// partial function
type int = string -> int

// total function
type tryParseInt = string -> Option<int>
```
Now we know for sure that `tryParseInput` will only produce `Some` number if the number was properly parsed. Otherwise, the function will return `None`. You can use the `tryParseInput` function in the `render` function as follows:
```fsharp
prop.onChange (tryParseInt >> Option.iter (SetNumberInput >> dispatch))
```
Now `SetNumberInput` will only be dispatched when we are able to parse the raw text from the input field. The lambda within `onChange` is just a fancy and more readable way of writing this:
```fsharp
prop.onChange (fun rawText ->
    match tryParseInt rawText with
    | Some number -> dispatch (SetNumberInput number)
    | None -> ())
```
This is the result of the application

<resolved-image source="/images/elm/int-no-exn.gif" />

Nice! No exceptions occur when the input is not a proper integer.


### Raw and Parsed: Extending The State

We were able to retrieve numeric input from the text box. But still the program feels a bit weird because when the input is invalid, the reflected text in the `span` element is not changed. Let's make it a bit nicer by showing the parsed number in green color if the input is actually an integer. Let's show an error message in red color if we were unable to parse the text.

<resolved-image source="/images/elm/int-validated.gif" />

To make this work, we need not only the parsed integer but also the raw text of the input in case we weren't able to parse it. Let's introduce a `Validated<'t>` type for this purpose with some helper functions:
```fsharp
type Validated<'t> =
    {  Raw : string
       Parsed : Option<'t> }

module Validated =
    let createEmpty() : Validated<_> =
        { Raw = ""; Parsed = None }

    let success raw value : Validated<_> =
        { Raw = raw; Parsed = Some value }

    let failure raw : Validated<_> =
        { Raw = raw; Parsed = None }
```
This `Validated<'t>` type contains both the `Raw` textual input *and* an optional parsed value. Because a parsed value may or may not be available, it is of type `Option<'t>`. Now we can model the state using this type:
```fsharp
type State = { NumberInput : Validated<int> }

type Msg =
    | SetNumberInput of Validated<int>

let init() = { NumberInput = Validated.createEmpty() }

let update msg state =
  match msg with
  | SetNumberInput numberInput ->
      { state with NumberInput = numberInput }
```
Now before we implement `render` we first need to rewrite `tryParseInt` to return `Validated<int>` instead of `Option<int>` as follows:
```fsharp
let tryParseInt (input: string) : Validated<int> =
    try Validated.success input (int input)
    with | _ -> Validated.failure input
```
The rest of the `render` function falls into place:
```fsharp
let validatedTextColor validated =
    match validated.Parsed with
    | Some _ -> color.green
    | None -> color.crimson

let render state dispatch =
  Html.div [
    prop.style [ style.padding 20 ]
    prop.children [
      Html.input [
        prop.valueOrDefault state.NumberInput.Raw
        prop.onChange (tryParseInt >> SetNumberInput >> dispatch)
      ]

      Html.h1 [
        prop.style [ style.color (validatedTextColor state.NumberInput) ]
        prop.text state.NumberInput.Raw
      ]
    ]
  ]
```
This implementation covers everything we need. The raw text is available for two purposes: to render the message on screen in the `Html.h1` element *and* to force the `Html.input` element to use it for the internal state using `valueOrDefault`. The result of the parsing is available to calculate the text color of the header element.

However, you might be thinking, "This is cool and all, but isn't it a bit overkill to map the raw value into a validated one if the whole purpose of this program is just to change the color of reflected text on the header?" And you be absolutely right!

If the goal was just to check whether the text is a valid number and changing the text accordingly, then we wouldn't need to keep track of the parsed value. Instead we could just keep track of the raw value and check whether it is parsable as an integer when rendering. However, this was just for demonstration purposes. In a real-world application, you *will* need the parsed value to use in subsequent events such as sending it to a web service.

Here I wanted to show you how you could make use of the F# type system to model validation and parsing correctly. Even though we were dealing with a simple integer, the ideas are the same when working with different data types and structures.

### Built-in HTML5 Number Validation

HTML input elements have a special attribute called `type`. This attribute tells the browser what type of input we are expecting from the user, whether it is textual (the default), numeric, boolean etc. We can specify this attribute by using the `type'` property. To tell the browser that the `Html.input` element is expecting a number from the user, we can specify it as follows:
```fsharp {highlight: [2]}
Html.input [
  prop.type'.number
  prop.valueOrDefault state.NumberInput.Raw
  prop.onChange (tryParseInt >> SetNumberInput >> dispatch)
]
```
Now the user can't type any non-numeric values into the input field and the browser adds custom validation such that the `onChange` does not get triggered when the input is not a proper number. This works even if you paste text from the clipboard. However, it is not enough because the browser seems to be happy to let an empty string through as a valid number. So you still have to do this validation business.

### Check Boxes

Another type of input element is a check box. Check box inputs correspond to boolean fields of the state. Assume we want to extend the example we have to the following sample application that implements a state toggle that will determine whether or not the resulting text from the text box is upper case:

<resolved-image source="/images/elm/checkbox-input.gif" />

Here, we want to keep track of whether the text should be capitalized or not. Therefore, we extend the state with a boolean field `Capitalized` with an initial value of `false`:
```fsharp {highlight: [3]}
type State = {
  TextInput: string
  Capitalized: bool
}

let init() = { TextInput = ""; Capitalized = false }
```
Next up is thinking of an event that changes the state. In the spirit of the `SetTextInput` event we can write a similar event called `SetCapitalized` that takes in a boolean value coming from the check box:

```fsharp {highlight: [3]}
type Msg =
  | SetTextInput of string
  | SetCapitalized of bool
```

Similarly, the `update` function follows the same pattern:
```fsharp
let update msg state =
  match msg with
  | SetTextInput name ->
      { state with TextInput = name }

  | SetCapitalized value ->
      { state with Capitalized = value }
```
Finally we have the `render` function:
```fsharp {highlight: ['16-23']}
let render state dispatch =
  Html.div [
    prop.style [ style.padding 20 ]
    prop.children [
      Html.input [
        prop.valueOrDefault state.TextInput
        prop.onChange (SetTextInput >> dispatch)
      ]

      Html.div [
        Html.label [
          prop.htmlFor "checkbox-capitalized"
          prop.text "Capitalized"
        ]

        Html.input [
          prop.style [ style.margin 5 ]
          prop.id "checkbox-capitalized"
          prop.type'.checkbox
          prop.isChecked state.Capitalized
          prop.onChange (SetCapitalized >> dispatch)
        ]
      ]

      Html.span (
        if state.Capitalized
        then state.TextInput.ToUpper()
        else state.TextInput
      )
    ]
  ]
```
Notice here the following:
 - A check box is simply an `Html.input` element with property `prop.type'.checkbox`.
 - The function `prop.isChecked` accepts a boolean input to set the value at start-up.
 - The event handler `onChange : bool -> unit` gives a nice way to retrieve the value.

Also there is the `span` as the last element where it renders its content based on whether the `Capitalized` field was set to true or not.

### Bootstrapping the `Program`

I have been skipping the last part of the applications where we bootstrap the Elmish program. As usual, we tie the triplet `init`, `update` and `render` into a program and start the application:

```fsharp
Program.mkSimple init update render
|> Program.withReactSynchronous "elmish-app"
|> Program.run
```

In the next section [React in Elmish](react-in-elmish) we will talk about what it means to use `React` as a rendering engine in an Elmish application and the consequences of this choice.
