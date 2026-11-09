+++
date = '2026-11-02T08:28:25-05:00'
draft = true
title = '2026 11 02 String_view Cpp vs Rust'
+++

Is C++ dead? Should all C++ start learning Rust? What about new developers?
Should they all just skip C++ and go straight to Rust?

No! I don't think this is a productive way to approach things. Languages
don't just die. COBOL is very much alive. So is C++.

A much more productive way to think, in my opinion, is to learn both and
learn of the advantages and disadvantages of each. For a C++ programmer,
learning Rust can definitely help: it can show blind spots and
deficiencies in C++. That is certainly my experience.

Where C++ relies on the programmer to not do BAD THINGS, Rust compiler
won't even compile. In today's post, I want to explain one such example.

## string_view

string_view type is a type for a non-owning view to a string. An object
of that type contains a pointer to a buffer of characters and a length.

Let's go through an example. Suppose we want a function that counts
words in a string. First, here's an implementation that gets the string
as an input argument of type string:

```
size_t count_words_string(const std::string& text) {
        bool in_word = false;
        size_t count{};
        for (auto ch : text) {
                if (std::isspace(ch)) {
                        if (!in_word) {}
                        else {
                                in_word = false;
                        }
                }
                else {
                        if (in_word) {}
                        else {
                                in_word = true;
                                ++count;
                        }
                }
        }
        return count;
}
```

Don't worry about the implementation. Let's look at the function signature:

    size_t count_words_string(const std::string& text)

The input argument has the type `const std::string&`. If we call the
function like this:

```
        std::string text = "This is some text.";
        auto count = count_words_string(text);
```

there is no copying of the buffer. The function really gets constant
reference to a string instance. So far, so good.

But what if we call the function like this:

```
        const char* text2 = "Another text buffer";
        count = count_words_string(text2);
```
?

Now on the caller side there is no instance of a string class, and
the function `count_words_string` is expecting an object of type
`const std::string&` as an argument. In this case, the compiler has to
create a temporary instance of `std::string` and call it constructor.
If the string (of type `const char*`) has 10 million characters, a new
temporary buffer has to be allocated and all the characters of the string
have to be copied. Then `count_words_string` is called, and after it
returns, the temporary string is deleted. In effect, it is something like
this:

```
		const char* text2 = "Aonther text buffer";
		const std::string tmp_string{text2};
		count = count_words_string(tmp_string);
		// Delete tmp_string here.
```

So this is obviously wasteful. C++17 introduced string_view, which
eliminates this wasteful construction and destruction of a temporary
string object. Our function, rewritten to use string_view, would look like
this:

```
size_t count_words(std::string_view text) {
        bool in_word = false;
        size_t count{};
        for (auto ch : text) {
                if (std::isspace(ch)) {
                        if (!in_word) {}
                                in_word = false;
                        }
                else {
                        if (in_word) {}
                        else {
                                in_word = true;
                                ++count;
                        }
                }
        }
        return count;
}
```

Now the input argument is `std::string_view text`. We don't have to make
it const, because string_view doesn't allow modifying the contents of the
buffer it points to. string_view has an overloaded constructor
that look like this:

```
constexpr string_view( const char* s )
```

and this constructor initializes and instance of a string_view by just
copying a pointer to string and calculating length.

Inside our new function, the code is completely the same. All the
functionality that we can use with a string, we can also use with
string_view - as long as we don't try to change the string.

## The Problem

So what's the problem? The problem is that C++ doesn't guarantee that
the buffer that string_view points to will exist as long as the
string_view instance exists. If the buffer is somehow deleted and
the instance of string_view is used after that, this is the dreaded
"undefined behavior".

Here's one simple example I could think of on the spot:

```
std::string_view select(uint8_t selection) {
        std::string s2_str{"second"};
        const std::string_view s1{"first"};
        const std::string_view s2{s2_str};

        switch (selection) {
        case 0:
                return s1;
        default:
                return s2;
        }
}
```

It is a contrived example, but I think it illustrates the problem well:
the function select returns a string_view that is supposed to contain
a pointer to a string buffer. But see what happens if selection=1:
it returns s2, which is a string_view initialized with s2_str. That
is a string that is destroyed just before the function returns, so
the return value is an invalid instance of string_view. As soon as
code tries to access that string, there is a crash.

How would we do this in Rust? Let's see the same function, translated to
Rust:

```
fn select<'a>(selection: u8) -> &'a str {                                      
    let s2_str = String::from("second");                                        
    let s1 = "first";                                                           
    let s2: &str = &s2_str;                                                     
    match selection {                                                           
        0 => s1,                                                                
        _ => s2,                                                                
    }                                                                           
}  
```

First, Rust requires us to add lifetimes to a reference that a function
returns, so this would not work:

    fn select(selection: u8) -> &str

Compiler complains:

```
error[E0106]: missing lifetime specifier
 --> prog.rs:7:30
  |
7 | fn select1(selection: u8) -> &str {
  |                              ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
```

So we add lifetime specifier, but even with that, the function doesn't compile:

```
error[E0515]: cannot return value referencing local variable `s2_str`
  --> prog.rs:11:2
   |
10 |       let s2: &str = &s2_str;
   |                      ------- `s2_str` is borrowed here
11 | /     match selection {
12 | |         0 => s1,
13 | |         _ => s2,
14 | |     }
   | |_____^ returns a value referencing data owned by the current function

error: aborting due to 1 previous error
```

That is precisely the problem in C++: we are allowed to work with references
in ways that can cause trouble and the compiler doesn't stop us. Rust won't
have it. How would we change that function in Rust to make the compiler happy?

There are two ways (that I can think of). The first way is change the signature
of the function so that it returns an owned value, in this case String:

```
fn select(selection: u8) -> String {                                         
    let s2_str = String::from("second");                                      
    let s1 = "first";                                                         
    let s2: &str = &s2_str;                                                   
    match selection {                                                         
        0 => String::from(s1),                                                
        _ => String::from(s2),                                                
    }                                                                         
}
```

Now, in order to return a String, we have to construct a String instead of
just returning a reference to a local object. This wouldn't compile:

    return s1;

It has to be transformed to this:

    return String::from(s1);

The other way to solve the problem would be to use Box. Box type is a way
to own a reference to an object on heap. Using Box, the function looks like
this:

```
fn select(selection: u8) -> Box<str> {
    let s2_str = String::from("second");
    let s1 = "first";
    let s2: &str = &s2_str;
    match selection {
        0 => Box::from(s1),
        _ => Box::from(s2),
    }
}       
```

This actually doesn't look that much different from the previous implementation
that uses String. Now, instead of String, we wrap a reference to str in a
box, and that box is an owned object whose ownership can be passed from the
function select to the calling function, which can decide what to do with it.
In any case, Rust will not let us hold a reference to an object that was
destroyed or, for that matter, forget a reference without destroying the object
it refers to.

That is something that makes writing memory-safe programs in C++ hard. In a
small function the have just looked at, it is easy to spot the problem. But when
we are dealing with a program that spans a thousand files with hundreds or
thousands of lines of code in each file, there is no way we can see the problem.
Even worse, the program may be correct until a programmer changes something and
inadvertently introduces a bug that causes the program to crash in a specific
circumstance.

