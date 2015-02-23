+++
date = "2015-02-22T21:31:35-05:00"
draft = true
title = "Don't be afraid to panic"
description = "Using panic and recover for clearer code"
tags = ["go", "programming"]
+++

In [Go][0], functions that can fail return an error value, that
will be non-nil if something went wrong. It is up to the caller of
a function to handle any errors that may occur. In the wild, such
Go code looks like this:

	x, err := strconv.Atoi(input)
	if err != nil {
		// handle or return the error
	}

While I have never found this too troublesome, if your most common
way to handle an error is by returning it, you will type more in
Go than you do in a language with exceptions, such as python.  There
are a few techinques described in the blog post "[Errors are values][1]"
that can help reduce some of the repetition.

Go has a seldom-used exception system in the form of three functions:
[defer, panic, and recover][2]. These three functions work together
to provide something like exceptions for Go:

- `defer()` ensures that a function is called when the current
  function is exited.
- `panic()` tells the Go runtime to begin unwinding the stack,
  calling deferred functions as as it exits each function.
- `recover()`, when called from within a `defer()`, stops any
  stack unwinding in process as the result of a panic.

It is not as flexible as the exception systems in other languages,
and is intended for truly exceptional errors, or programmer mistakes,
where the best response is to crash the program. However, sometimes
the behavior of `defer`, `panic`, and `recover` can be useful for handling
more routine errors. When developing the [xsd][3] package, I defined
a `walk` function to walk a tree of xml elements:

	func (el *xmltree.Element) Walk(func (*xmltree.Element) error) error

I quickly found that every time I encountered an error, I would
immediately stop parsing and "bubble up" the error, adding annotations
along the way:
	
	var result []Element
	err := root.Walk(func(el *xmltree.Element) {
		...
		var v Element
		max := el.Attr("maxOccurs")
		if max == "unbounded" {
			v.Plural = true
		} else if max != "" {
			i, err := strconv.Atoi(max)
			if err != nil {
				return fmt.Errorf("Invalid maxOccurs %q: %v",
					max, err)
			}
			v.Plural = (i > 1)
		}
		result = append(result, v)
		return nil
	})
	return result, err

I soon found the repetitive error checking overwhelming. I quickly rewrote
the code to use `panic`.

	func stop(msg string) {
		panic(parseError{message: msg})
	}
	
	func walk(root *xmltree.Element, fn func(*xmltree.Element)) {
		defer func() {
			if r := recover(); r != nil {
				if err, ok := r.(parseError); ok {
					err.path = append(err.path, root)
					panic(err)
				} else {
					panic(r)
				}
			}
		}()
		for i := 0; i < len(root.Children); i++ {
			fn(&root.Children[i])
		}
	}
	
	// defer catchParseError(&err)
	func catchParseError(err *error) {
		if r := recover(); r != nil {
			*err = r.(parseError)
		}
	}

I then wrote several wrappers around library functions that would panic
instead of returning an error:

	func parseInt(s string) int {
		switch s {
		case "":
			return 0
		case "unbounded":
			return -1
		}
		n, err := strconv.Atoi(s)
		if err != nil {
			stop(err.Error())
		}
		return n
	}

The result was less cluttered parsing code:
	
	var result []Element
	walk(root, func(el *xmltree.Element) {
		var v Element
		if max := parseInt(el.Attr("maxOccurs")); max < 0 || max > 1 {
			v.Plural = true
		}
		result = append(result, v)
	})
	return result

This works because in this use case, all errors are handled in the
same way, and because recursion is heavily used while parsing due
to the deep nesting present in XML schema documents. Because all
of the parsing functions went from returning a result and an error,
to returning a single result, *the functions became much more
composable*, almost like commands in a unix pipeline. Compare:

	// With explicit error returns
	elem, err := parseElement(root)
	if err != nil {
		return fmt.Errorf("Error parsing element %s: %v", root.Name.Local, err)
	}
	t.Elements = append(t.Elements, elem)
	
	// With "exceptions"
	t.Elements = append(t.Elements, parseElement(root))

While the conventional advice around `defer`, `panic` and `recover`
is not to use them, recognizing cases where they are appropriate
can lead to clearer and more concise code.

[0]: http://golang.org
[1]: http://blog.golang.org/errors-are-values
[2]: http://blog.golang.org/defer-panic-and-recover
[3]: http://godoc.org/aqwari.net/xml/xsd
