---
external: false
draft: false
title: Sealed classes in dart 3
description: Unlock the power of sealed classes in Dart! Dive deep into the world of subtype handling even before Dart 3's direct support. Explore how packages like Freezed simplify the process and discover the benefits of sealed types. This comprehensive guide walks you through the basics, internals, and real-life applications of sealed classes in Dart, complete with hands-on code examples. Whether you're a beginner or a seasoned developer, learn how to leverage sealed classes for cleaner and more efficient code
date: 2023-06-05
---

# Sealed classes

Sealed classes are very useful when dealing with subtypes. In Dart 3, it will be directly supported in the language. Until then, packages like Freezed has made our lives easier by providing sealed types.

In this post, we are going to learn what benifits are there from sealed types and how we can utilize it, in dart with some real life examples.

### Basics

Let’s start off with the basic syntax on how we can declare sealed classes.

```dart
sealed class Animal {}
```

We can create subtypes by just extending this class

```dart
class Dog extends Animal {}

class Cat extends Animal {}
```

switch statements supports `exhaustiveness` so no subtypes of sealed class are missed.

```dart
switch(animal){
	case Dog(): print("Bark!");
	break;
	case Cat(): print("Meow");
	break;
}
```

### Internals

Sealed classes put two restrictions on us:

- Sealed classes are internally abstract, so we cannot create an object of them.
- We need to declare the subtypes of the sealed class in the same library.

Due to these constraints, we end up getting two major benefits:

- All the subtypes are easily found and enumerated.
- For `exhaustiveness`, all the subtypes need to be a child of at least one of the subtypes of the sealed class.

### Real life example

Now we are going to see real life examples of how we can use sealed classes in our daily life. We are going to make a todo app with json placeholder api. You can find the source code here.

To start off, we will make a file `endpoints.dart` and store the url here.

```dart
const todosUrl = 'https://jsonplaceholder.typicode.com/todos';
```

In this section, we will be creating our first sealed class called `Either`. This class is widely used in basic functional programming, and is used to handle error cases. The `Either` class is defined as a type that can hold either a successful value or a failure value. This can be useful in situations where there may be multiple error cases, and we want to handle them all in a clean and easy-to-understand way.

For those who are not familiar with functional programming, `Either` is a type that represents the concept of choice between two values. It can be used to handle error cases, or to create a choice between two values in a more straightforward manner. If you are interested in learning more about `Either`, you can read about it [here](https://codewithandrea.com/articles/functional-error-handling-either-fpdart/).

To implement the `Either` class, we will not be using any external packages. Instead, we will be implementing it ourselves, and then see how we can use it later on in this blog. This will give us a better understanding of how the `Either` class works, and how it can be used in different situations.

```dart
/// [Either] is a type that can be either [Left] or [Right].
/// It is used to represent a value of one of two possible types (a disjoint union).
/// It has two generic type parameters [K] and [V].
sealed class Either<K, V> {}

/// [Left] is a type of [Either] that can hold a value of type [K].
class Left<K, V> extends Either<K, V> {
  final K value;

  Left(this.value);
}

/// [Right] is a type of [Either] that can hold a value of type [V].
class Right<K, V> extends Either<K, V> {
  final V value;

  Right(this.value);
}

/// [left] is a helper function that returns an instance of [Left].
Either<K, V> left<K, V>(K value) => Left(value);

/// [right] is a helper function that returns an instance of [Right].
Either<K, V> right<K, V>(V value) => Right(value);
```

We will now create a repository class named `TodoRepository` that fetches data from the API. It simply makes a `GET` call, checks for errors (if any), and returns the result as an `Either` type.

```dart
class TodoRepository {
  final http.Client client = http.Client();

  /// Fetch todos from the server. Uses the `http` package.
  /// [Either] type which we created previously is used to handle errors.
  /// [K] is [Failure] and [V] is [List<Todo>].
  Future<Either<Failure, List<Todo>>> fetchTodos() async {
    try {
      final response = await client.get(Uri.parse(todosUrl));
      if (response.statusCode == 200) {
        final todos = (json.decode(response.body) as List)
            .map((e) => Todo.fromJson(json.encode(e)))
            .toList();
        return right(todos);
      } else {
        return left(Failure("Couldn't find the todo"));
      }
    } on SocketException {
      return left(Failure("No Internet Connection"));
    } on HttpException {
      return left(Failure("Couldn't find the todo"));
    } on FormatException {
      return left(Failure("Bad response format"));
    } catch (e) {
      return left(Failure(e.toString()));
    }
  }
}
```

This outlines how we'll fetch data from the internet. Next, we'll create notifiers to emit the appropriate state. We'll be implementing a functionality similar to Riverpod's `[AsyncValue](https://pub.dev/documentation/riverpod/latest/riverpod/AsyncValue-class.html)`, which is a utility for safely manipulating asynchronous data. However, we'll be doing so without using Riverpod. 

Our custom class, `FutureValue`, will serve the same purpose and ensure we don't miss any possible state of an asynchronous operation through `exhaustiveness` in switch cases.

Our `FutureValue` class looks like this

```dart
/// FutureValue is a sealed class that can be used to represent the state of a
/// Future. It has three subtypes: [Loading], [Success], and [Error].
sealed class FutureValues<T> {}

/// [Loading] is used to repsent the state of a Future when it is in progress.
class Loading<T> extends FutureValues<T> {}

/// [Success] is used to repsent the state of a Future when it is completed
class Success<T> extends FutureValues<T> {
  final T value;

  Success(this.value);
}

/// [Error] is used to repsent the state of a Future when it has failed.
class Error<T> extends FutureValues<T> {
  final String message;

  Error(this.message);
}
```

Now let’s write the `notifier` which will be responsible for managing the state of our request.

```dart
class TodoNotifier extends ValueNotifier<FutureValues> {
  /// Inject [TodoRepository] in the constructor.
  /// calls [super] with [Loading] as the initial value.
  TodoNotifier(this._todoRepository) : super(Loading());

  final TodoRepository _todoRepository;

  /// Fetch todos from the repository.
  /// [Either] type which we created previously is used to handle errors.
  /// changes [value] to [Loading] when the request is in progress.
  /// changes [value] to [Success] with the list of todos when the request is successful.
  /// changes [value] to [Error] with the error message when the request fails.
  void getTodos() async {
    value = Loading();
    final todoStatus = await _todoRepository.fetchTodos();
    switch (todoStatus) {
      case Right():
        value = Success(todoStatus.value);
        break;
      case Left():
        value = Error(todoStatus.value.message);
        break;
    }
  }
}
```

With this, our data part of the app is completed. All we need now is a UI to display the list of todo. Let’s create a class called `TodoScreen` and use `TodoNotifier` to show all the todos.

```dart
class TodoScreen extends StatefulWidget {
  const TodoScreen({super.key});

  @override
  State<TodoScreen> createState() => _TodoScreenState();
}

class _TodoScreenState extends State<TodoScreen> {
  final _todoNotifier = TodoNotifier(TodoRepository());
  @override
  void initState() {
    _todoNotifier.getTodos();
    super.initState();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text("Dart 3 : Todo app"),
      ),
      body: ValueListenableBuilder<FutureValues<List<Todo>>>(
          valueListenable: _todoNotifier,
          builder: (context, value, child) {
            // switch will automatically ask you to cover all the cases of [FutureValues]
            switch (value) {
              // This is how we can use pattern matching in Dart, will be covered in coming blogs.
              // Loading() => _buildLoading(),
              // Error(message: var error) => _buildError(error),
              // Success(value: var value) => _buildSuccess(value),
              case Loading():
                return _buildLoading();
              case Error():
                return _buildError(value.message);
              case Success():
                return _buildSuccess(value.value);
            }
          }),
    );
  }

  Widget _buildLoading() {
    return const Center(
      child: CircularProgressIndicator(),
    );
  }

  Widget _buildError(String message) {
    return Center(
      child: Text(message),
    );
  }

  Widget _buildSuccess(List<Todo> todos) {
    return ListView.builder(
      itemCount: todos.length,
      itemBuilder: (context, index) {
        return ListTile(
          title: Text(
            todos[index].title,
            style: TextStyle(
              decoration: todos[index].completed
                  ? TextDecoration.lineThrough
                  : TextDecoration.none,
              color: todos[index].completed ? Colors.green : Colors.black,
            ),
          ),
        );
      },
    );
  }
}
```

With this, we come to the end of the blog and we have covered how to create `sealed` classes in dart and how to use them. We also covered some examples on how it can be used in real life.