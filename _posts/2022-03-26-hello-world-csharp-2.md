---
title: "Hello, World! (C#) Part 2"
subtitle: 
toc: true
toc_sticky: true
toc_icon: code
---

If you haven't already, do [Hello World Part 1](/hello-world-csharp/).

## End Result
A simple console app that takes some user input and displays different things depending on that input.

## User Input
When we left off, we had a console app that said "Hello, World!" in the console.  No matter how many times you run it, the same thing will happen.

*Pretty boring*, right?

Most computer programs take some sort of input so that they can do *different* things depending on that input.  So let's get some user input!  If you think back to last time, we used `Console.ReadLine()` to stop the program from closing until you hit Enter.  We changed it to `ReadKey()` so any key would work, but let's change it back and try something out.  Your code should look like this:

```csharp
Console.WriteLine("Hello, World!");
Console.WriteLine("Press any key to continue...");
Console.ReadLine();
```

Run your code, and instead of immediately hitting Enter, try typing things in first and *then* hitting Enter.  You may be wondering, *"where did that input go?"*  Well, we're about to find out!

## Return Values

As it turns out, every statement in code has a *return value*.  Think of like this: every action has a *reaction*.  So when you call `ReadLine()` it returns a value.  In Visual Studio, if you hover over `ReadLine()` you'll see the word `string` appear next to the name.

A `string` is a "string of characters" - this could be a word, a sentence, a whole paragraph, a single letter, or nothing at all (i.e.: an *empty* string).  So when you type things into the console and hit Enter, `ReadLine()` gives the string of characters you typed back to the code.

*Yeah, but where did it go?*

Well, nowhere... yet!

## Variables

To keep track of the response, we need to store it.  This is what **variables** are for.  Remember algebra?  Things like `2x + 5y = 3z`?  `x`, `y`, and `z` are variables - they're basically just placeholders that could represent any number.  We can make a variable in code for the same reason, but it can store any type of data - not just numbers.  In this case, we need to make a `string` to store the `string` response from `ReadLine()`.  We do it like this:

```csharp
string result = Console.ReadLine();
Console.WriteLine(result);
Console.ReadKey();
```

The first line does a few things: 
- `string` specifies the type of the variable
- `result` is the name of the variable.  It has to start with a letter and can't have spaces, but can otherwise be whatever you want!
- `=` isn't exactly what it meant in algebra.  It means "put the value on the right into the variable on the left".

Altogether, it is taking the value from the right side (i.e.: `Console.ReadLine()`) and assigning that into the variable on the left side (i.e.: `result`).  The next line is writing the `result` variable back out to the console so we can see it.  Finally, we `ReadKey()` so the console won't automatically close.  Run your code a few times and try typing different things (or nothing!) and see what happens.

## Concatenation
Let me guess: you're pretty underwhelmed by the above example.  Let's do something more than just print the same variable right back out!

Here's the idea: the program will ask for your name and then say "Hello, Noah!" instead of "Hello, World!".  First, let's give our user a prompt:

```csharp
Console.Write("Enter your name: ");
string name = Console.ReadLine();
```
Notice I used `Write` rather than `WriteLine` so that their input will show up on the same line!  Now, let's say hello:

```csharp
Console.WriteLine("Hello, " + name + "!");
```
Here, the `+` serves as the "concatenation" operator.  Concatenation is a fancy way of saying "mash these two `string`s together.  Run your code and type your name to see what happens!

Let's go one step further and ask for your *last* name and print it out as well!  See if you can figure it out yourself before scrolling down, but the final product will look like this:

```csharp
Console.Write("Enter your first name: ");
string firstName = Console.ReadLine();
Console.Write("Enter your last name: ");
string lastName = Console.ReadLine();
Console.WriteLine("Hello, " + firstName + " " + lastName + "!");
```
It works, but I bet you can imagine how gross it would look if we added a middle name, suffix, etc.  Could there be a better way... 🤔

## Prettier Concatenation
A while back, Microsoft added "string interpolation" to C#'s bag of tricks.  That's a fancy way of saying that there's now a way to mix variables right into your strings, and all you need is a dollar sign `$` and some curly braces `{}`.  We can change our `WriteLine` to look like this:

```csharp
Console.WriteLine($"Hello, {firstName} {lastName}!");
```
The `$` lets C# know you want to use string interpolation, and the curly braces let C# know where the variables start and end.  Try running it with and without the `$` and see what happens.

## Conditionals
Have you tried hitting enter without giving a response?  What happens?  Did you get a message like `Hello,  !`?  Let's put some logic in so that our user *has* to enter something.

What we need to know is `if` the user entered something or not.  `if` the input was blank, we need to ask for their first name again.  We can do it like this:

```csharp
Console.Write("Enter your first name: ");
string firstName = Console.ReadLine();
if (firstName == "")
{
    Console.Write("Enter your first name: ");
    firstName = Console.ReadLine();
}
```

Notice the `if` statement.  It looks at what's in the parentheses after it to see if it's `true` or `false`.  In those parentheses you'll see `firstName == ""` - notice the double equals `==`.  Like we learned above, a single equals `=` stuffs a value into a variable.  A double equals `==` does a comparison.  If the things on each side of `==` are equal, it will evaluate to `true`.  Otherwise, it evaluates to `false`.

**QUICK ASIDE**: You may have noticed that `string` is missing next to `firstName` inside of the `if` statement.  That's because we only need to specify the type when a variable is first defined.  `firstName` already exists above, so we don't need to define it again.  Try putting `string` there and you'll see that Visual Studio stops you, saying that name is already used!  Moving on:

If the statement in `if`'s parentheses evaluates to `true`, it will run the commands in its curly braces `{}`.  If it is `false`, those commands get skipped.  Run your new code and see what happens if you do or do not give a first name.

Did you try giving a bad name more than once?  It proceeds on to the next part.  Why?  Because `if` only checks things *once*.  If we want it to check twice, we would need *two* `if` statements.  But we don't know how many times our user will enter bad input, so we need to try something else.

## Loops
What we need to do is *keep on checking* until we get good input.  In other words, `while` the input is bad, we should ask for it again.  So make one teeny tiny change - instead of `if`, use `while`:

```csharp
while (firstName == "")
{
    Console.Write("Enter your first name: ");
    firstName = Console.ReadLine();
}
```
Run your code and check it out.  **Hooray!**  Before we do the same thing for `lastName`, look at the code again.  Notice how we have copy/pasted the exact same lines?  As a general rule of thumb, if you are copy/pasting code like that, there's a better way of doing things.  In this case, we can remove the first time we prompt for the user's name and instead do this:

```csharp
string firstName = "";
while (firstName == "")
{
    Console.Write("Enter your first name: ");
    firstName = Console.ReadLine();
}
```
Let's think about what's happening here.  First, we put an empty string `""` into our variable `firstName`.  We then check to see if it equals `""` - of course it does, since we just set it to that value.  That guarantees that we'll enter the `while` loop at least once.  Now our code is just a little less repetitive.

Do the same thing for `lastName` - set it to `""` and check it in a `while` loop.  When you're done, your program should keep asking for your first name, then keep asking for your last name, then display that name.

That's all for now!  Next time, we'll look at variables of different types, a loop other than `while`, and some helpful things `string` can do for us.  For now, congratulate yourself, because:
> **We made a program**! 🎉