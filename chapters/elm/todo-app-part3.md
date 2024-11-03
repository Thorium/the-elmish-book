# To-Do List Application: Part 3

In this section, we will add a feature to our To-Do list application that makes every item in the list editable. It looks as follows:

<div style="width:100%">
  <div style="margin: 0 auto; width:65%;">
    <resolved-image source="/images/elm/todo-part3.gif" />
  </div>
</div>

Notice here that we have two requirements as well:
 - Edit one To-Do item at a time.
 - Being able to cancel the editing mode leaving the item unaltered.

You can see and use the application [live here](https://zaid-ajaj.github.io/elmish-todo-part3/).

### Modelling the State

Alright, this one can be a bit tricky. This is because this "editing mode" is not always active. The user has to click the edit button for a certain To-Do item, then the edit form is shown for that specific item. We know at least that we have to maintain the `Id` of the To-Do item being edited. Let's start with that:
```fsharp {highlight: [10]}
type Todo = {
  Id : int
  Description: string
  Completed: bool
}

type State = {
  NewTodo : string
  TodoList : Todo list
  TodoBeingEdited : int
}
```
Here I have added `TodoBeingEdited` which is an `int` to keep track of the To-Do being edited. But we said that this editing mode is not always active and certainly when the application starts, there is no To-Do item being edited yet.

A better way to model information that is not available at application startup is to use the `Option<'t>` type such that when we initialize the state, we can say: "This piece of information is optional and can be missing. You can't assume the presence of the value and you have to check it first".

```fsharp {highlight: [10]}
type Todo = {
  Id : int
  Description: string
  Completed: bool
}

type State = {
  NewTodo : string
  TodoList : Todo list
  TodoBeingEdited : int option
}
```
Where `TodoBeingEdited` will be `None` when initializing the state.

You might argue that using `Option<int>` is not necessary for initializing the state and that one could just use a useless value like `-1` for the To-Do item being edited. The thing is, the initial state is *not* our only concern. We use this value both inside the `update` and the `render` function. Modelling the field as optional forces us *at compile time* to check whether or not we can use the value. It is all about what you can or cannot assume when updating the state or rendering the user interface. The fields used in `State` should convey your *intentions* so that your colleagues or anyone for that matter can understand what your code is supposed to do.

Suppose I was going through your code and I read `TodoBeingEdited : int`. I would be confused because the code assumed there is always a To-Do item being edited. Doesn't make sense. However, if I read `TodoBeingEdited : int option` I immediately know this is something that can be missing. I should take into account both the potential presence and potential absence of the value. This also makes sense from the UI point of view because a user is not always editing an item.

OK, now that we have that out of the way, we ask ourselves, "Is this all the information needed for editing a To-Do item, both for the `update` and the `render` function?"

Well, if we know which item is being edited, then the edit form can simply start modifying the description of said item as the user is typing. But unfortunately this goes against the requirement of being able to cancel the editing mode without changing the contents of the item. This is because when we cancel the edit mode, the To-Do item should retain it's original description, not the one that was typed in during the edit mode. Now we know: we must keep track of the new edited description *separately* from the description of the item being edited. Let's incorporate our findings into the `State` type, introducing a new model called `TodoBeingEdited`. This new model represents the required information to support editing a To-Do:

```fsharp {highlight: ['7-10', 15]}
type Todo = {
  Id : int
  Description : string
  Completed : bool
}

type TodoBeingEdited = {
  Id: int
  Description: string
}

type State = {
  TodoList: Todo list
  NewTodo : string
  TodoBeingEdited : TodoBeingEdited option
}
```

Now when in editing mode, we have to know which item we are editing (which `Id`) and also the new description for that item. If the user applies the edits, the description of the item being edited is replaced with the description that the user provided. Otherwise, when the user cancels the editing mode, we just reset `TodoBeingEdited` back to `None` like it initially was on application startup.

Moving on to the events of the application: the `Msg` type. We will keep the events and their implementation from Part 2 as is because they should keep doing what they are doing. We will be adding a couple more, namely:

```fsharp {highlight: ['6-9']}
type Msg =
  | SetNewTodo of string
  | AddNewTodo
  | DeleteTodo of int
  | ToggleCompleted of int
  | CancelEdit
  | ApplyEdit
  | StartEditingTodo of int
  | SetEditedDescription of string
```
Here I have added four new events, three of which will be associated with buttons:
 - `StartEditingTodo of int` starts the edit mode for a specific To-Do item when you click the edit button.
 - `CancelEdit` cancels the editing mode by resetting the `TodoBeingEdited` back to `None`.
 - `ApplyEdit` takes the information from `TodoBeingEdited` (if it is available!) and applies the changes to the `Todo` item from `state.TodoList`, resetting the edit form to `None`.
 - `SetEditedDescription of string` modifies the edited description in `TodoBeingEdited` (if it has a value).

### Implementing State Updates

The implementation of these events is quite straightforward:

```fsharp
| StartEditingTodo todoId ->
    let nextEditModel =
      state.TodoList
      |> List.tryFind (fun todo -> todo.Id = todoId)
      |> Option.map (fun todo -> { Id = todoId; Description = todo.Description })

    { state with TodoBeingEdited = nextEditModel }

| CancelEdit ->
    { state with TodoBeingEdited = None }

| ApplyEdit ->
    match state.TodoBeingEdited with
    | None -> state
    | Some todoBeingEdited when todoBeingEdited.Description = "" -> state
    | Some todoBeingEdited ->
        let nextTodoList =
          state.TodoList
          |> List.map (fun todo ->
              if todo.Id = todoBeingEdited.Id
              then { todo with Description = todoBeingEdited.Description }
              else todo)

        { state with TodoList = nextTodoList; TodoBeingEdited = None }

| SetEditedDescription newText ->
    let nextEditModel =
      state.TodoBeingEdited
      |> Option.map (fun todoBeingEdited -> { todoBeingEdited with Description = newText })

    { state with TodoBeingEdited = nextEditModel }
```
Notice here the use of `Option.map` and `List.tryFind` when working with `state.TodoBeingEdited`. The code takes into account the fact that the value can be missing and proceeds accordingly to work with the value only if it is present. The result? No *implicit* assumptions! Our intention is right there translated in the code.

### Rendering the User Interface

Back to the `render` function. This time it is actually more interesting because here we must show the edit form while we are in the edit mode and the To-Do item otherwise. I left the function `renderTodo` from Part 2 as is, except for the edit button that starts the edit mode. To render the edit form (text box, save and cancel button) I have added another function called `renderEditForm`. The root `render` function decides which of these to call when rendering each todo as follows:
```fsharp {highlight: [1, '11-15']}
let renderEditForm (todoBeingEdited: TodoBeingEdited) (dispatch: Msg -> unit) =
  (* . . . *)

let renderTodo (todo: Todo) (dispatch: Msg -> unit) =
  (* . . . *)

let todoList (state: State) (dispatch: Msg -> unit) =
  Html.ul [
    prop.children [
      for todo in state.TodoList ->
        match state.TodoBeingEdited with
        | Some todoBeingEdited when todoBeingEdited.Id = todo.Id ->
            renderEditForm todoBeingEdited dispatch
        | otherwise ->
            renderTodo todo dispatch
    ]
  ]

let render (state: State) (dispatch: Msg -> unit) =
  Html.div [
    prop.style [ style.padding 20 ]
    prop.children [
      appTitle
      inputField state dispatch
      todoList state dispatch
    ]
  ]
```
The highlighted part is very important. For every item in `state.TodoList` we check whether we are editing *that* item. If that is the case, we render the "edit form". We do so by testing whether `state.TodoBeingEdited` has a value and that value (of `todoBeingEdited`) has an `Id` equal to the `Id` of the item we are rendering. For all other cases, simply render the `todo`.

Also notice that every "smaller part" of the root `render` function (the functions `renderTodo` and `renderEditForm`) only takes in the *minimal* amount of information needed. This makes it easier to think about these parts in separation. It also avoids information redundancy. The implementation of both functions is trivial as well. First, the new `renderEditForm` function:
```fsharp
let renderEditForm (todoBeingEdited: TodoBeingEdited) (dispatch: Msg -> unit) =
  div [ "box" ] [
    div [ "field"; "is-grouped" ] [
      div [ "control"; "is-expanded" ] [
        Html.input [
          prop.classes [ "input"; "is-medium" ]
          prop.valueOrDefault todoBeingEdited.Description;
          prop.onTextChange (SetEditedDescription >> dispatch)
        ]
      ]

      div [ "control"; "buttons" ] [
        Html.button [
          prop.classes [ "button"; "is-primary"]
          prop.onClick (fun _ -> dispatch ApplyEdit)
          prop.children [
            Html.i [ prop.classes ["fa"; "fa-save" ] ]
          ]
        ]

        Html.button [
          prop.classes ["button"; "is-warning"]
          prop.onClick (fun _ -> dispatch CancelEdit)
          prop.children [
            Html.i [ prop.classes ["fa"; "fa-arrow-right"] ]
          ]
        ]
      ]
    ]
  ]
```
Then, `renderTodo` is the same as from part 2 but with an addition of the button that turns the items into the edit mode:
```fsharp {highlight: ['21-27']}
let renderTodo (todo: Todo) (dispatch: Msg -> unit) =
  div [ "box" ] [
    div [ "columns"; "is-mobile"; "is-vcentered" ] [
      div [ "column" ] [
        Html.p [
          prop.className "subtitle"
          prop.text todo.Description
        ]
      ]

      div [ "column"; "is-narrow" ] [
        div [ "buttons" ] [
          Html.button [
            prop.classes [ "button"; if todo.Completed then "is-success"]
            prop.onClick (fun _ -> dispatch (ToggleCompleted todo.Id))
            prop.children [
              Html.i [ prop.classes [ "fa"; "fa-check" ] ]
            ]
          ]

          Html.button [
            prop.classes [ "button"; "is-primary" ]
            prop.onClick (fun _ -> dispatch (StartEditingTodo  todo.Id))
            prop.children [
              Html.i [ prop.classes [ "fa"; "fa-edit" ] ]
            ]
          ]

          Html.button [
            prop.classes [ "button"; "is-danger" ]
            prop.onClick (fun _ -> dispatch (DeleteTodo todo.Id))
            prop.children [
              Html.i [ prop.classes [ "fa"; "fa-times" ] ]
            ]
          ]
        ]
      ]
    ]
  ]
```

We have reached the end of this part of our To-Do application. You can check out the [source code here](https://github.com/Zaid-Ajaj/elmish-todo-part3) for reference.

I hope at this point that you have developed an idea of how an Elmish program works, how to think about a requirement in terms of state transitions, and how the application evolves as we add more features to it. In the next section, I want to put what you have learned to the test! Let's jump right in.
