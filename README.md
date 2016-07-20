[![Build Status](https://api.travis-ci.org/tkrajina/ftmpl.svg)](https://travis-ci.org/tkrajina/ftmpl)

# FTMPL: fast (and typesafe) templating for Golang

ftmpl is a fast/compiled/typesafe templating "language" for golang.

## Why ftmpl?

 * It's faster (run `go test -v . -run=TestComparisonWithGolangTemplates` for a comparison). The builtin templates involve a lot of reflection, ftmpl "compiles" to simple Golang functions.
 * No need to learn another templating "language", ftmpl is just plain Go
 * Type safety (why have a typesafe language and a nontypesafe templating engine?)
 * Base/extended templates

## Installation

    go get github.com/tkrajina/ftmpl

## Templating files and functions

Every template must be saved in a file with extension `.tmpl`.
The template file can be "compiled" in Go code (and this is typically done in the build procedure).

If the file is named `MyTemplate.tmpl` then the `ftmpl` utility will produce a function `TMPLMyTemplate()`.

Compiling files is done my invoking:

    ftmpl -targetgo target_dir/templates_generated.go source_dir

The resulting code is pretty straightforward, and you will probably need to check (but not edit!) it in case of errors.

For template examples see [here](example/) (`.tmpl` files).
For the "compiled" golang code from those examples [see here](example/compiled.go). The resulting code is always formatted using `go fmt`.

If you prefer using `go generate` add a comment...

    //go:generate ftmpl -targetgo target_dir/templates_generated.go source_dir

...anywhere in your code.

The prefix `TMPL` in the function name can be changed by a cmd line switch.

### Watch and recompile

`ftmpl` can "watch" for changes in your source directory. When any of the `*.tmpl` files change there, it will "recompile" all the templates. Just add `-watch` when calling it:

    ftmpl -watch -targetgo target_dir/templates_generated.go source_dir

If you start multiple `ftmpl` "watch" processes, you can stop them all with:

    ftmpl -unwatchall

## Template files

An example template `ShowTime.tmpl`

    !#import "time"
    <html>
        <body>
            It's {{s time.Now().String() }}
        </body>
    </html>

It will be compiled to the function `func TMPLShowTime()`

## Templates with arguments

If you want your template function to have arguments:

    !#import "time"
    !#arg t time.Time
    <html>
        <body>
            It's {{s t.String() }}
        </body>
    </html>

Now the compiled function will be `func TMPLShowTime(time.Time)`

## Placeholders

Placeholders are expressions to be executed and written when calling the template. 
Since ftmpl is a typesafe templating it needs to know of which type is the expression, and based on that to properly format it:

Use:

 * `{{s expression }}` for string expressions
 * `{{d expression }}` for number expressions
 * `{{t expression }}` for bool expressions
 * ...

It's simple, when compiled `{{s expression }}` will end up as `fmt.Sprintf("%s", expression)`, `{{v expression }}` as `fmt.Sprintf("%v", expression)`, etc.

`{{ expression }}` is the same as `{{v expression }}`.

For more complex formatting, prefix with `%`:

 * `{{%5.5f 1.222222222 }}` is equivalent to `fmt.Sprintf("%5.5f", 1.222222222)`
 * `{{% 15f "padded" }}` is equivalent to `fmt.Sprintf("% 15f", "padded")`

### Escape and unsecape

If you use `{{s expresssion }}` the result will be escaped using `html.EscapeString()`. If you want to write *the exact same string* (without escaping), use `{{=s expression }}`.

**Important** that `{{v expression }}` *is not escaped* even when the resulting expression is a string!

## Code

There are two ways to write go code in templates:

### Lines starting with "!"

    !#arg n int
    <html>
        <body>
    !for i := 0; i < n; i++ {
            i is now {{d i }}
    !}
        </body>
    </html>

Basically every line starting with `!` (but not `!#`) will be translated as go code in the resulting function.

If you (really?) need a non-golang line starting with `!` start it with `!!`.

### Embedded code

    !#arg n int
    <html>
        <body>
            {{! for i := 0; i < n; i++ { }}i is now {{d i }}{{! } }}
        </body>
    </html>

...so by using {{! code }} you embed a go command in that place.

Note that (unlike some other templating languages) the `{{! code }}` notation *is not multiline*. It must contain only one go command/line.

For `for` and `if` control structures you can ommit `{`, `}` or `} else {` (open and close block of code).
Use `end` to close and `else` (with `if`) instead:

For example `if`:

    {{! if n > 0 }}{{d n}} biger than 0{{! end }}
    {{! if n > 5 }}{{d n}} biger than 5{{! else }}{{d n}} smaller than 5{{! end }}

With `for`...

    {{! for i := 0; i < n; i++ }} i={{d i }} {{! end }}

...or as `!` lines:

    !for i:=0; i < 5; i++
        i is now {{d i }}
    !end

## Directives

Like Go code, special "directive" commands like `!#arg` or `!#import` can also be used as lines starting with `!#` or embedded in the code: `{{!#import "time" }}`, `{{!#return }}`, ...

## Exit from template

To prematurely end the execution with an error (depending on an expression), you can use:

    !#errif <expression>, "<error_message>"

...or...

    !#return

## Base templates and extensions

When you need a base website template, and pages "extending it":

    !#arg title string
    <html>
        <head>
            <title>{{s title }}</title>
    !#include head
        </head>
        <body>
    !#include body
    !#include footer
        </body>
    </html>

And then a typical page will be something like:

    !#extends base
    !#arg something int

    !#sub head
    <script>
    alert("included")
    </script>

    !#sub body
    <h1>Body!</h1>

If the name of the extended template is `MyPage.tmpl` then the resulting template function will have both the arguments from *that template* and arguments from the *base template*: `func TMPLMyPage(title string, something int)`.

Typically you will have many template arguments, so the best way to deal with them is to pack them all into one "base page structure" and another struct for every page.
The function will then be `func TMPLMyPage(baseParams BasePageParams, pageParams MyPageParams)`

If a `!#sub` chunk is declared in the extended template, but not in the base -- `ftmpl` will show you a warning during compilation, but it will be present int the compiled code.

## Inserting (sub)templates

Sometimes you need some templates just "copied" in other templates. You can do that with `!#insert`. For example:

    !#arg a int
    Will insert something here: {{!#insert "insertion.tmpl" }}

The `insertion.tmpl` template file:

    !#nocompile
    a={{d a }}

Is equivalent to:

    !#arg a int
    Will insert something here: a={{d a }}

Note that the `insertion.tmpl` alone is an **invalid** template because it contains a previously undeclared variable `a`. But, when inserted, then the variable `a` is OK. That template makes sense only when included in another template. That's why it has `!#nocompile` in it.

Another way to do inserts is to just call the generated template function:

    !#arg a int
    Will insert something here: {{=s TMPLinsertion(a) }}

In this case `insertion.tmpl` must **not** have `!#nocompile`, and the variable `a` must be declared with `!#arg a int`.

## No compile

Some template files make no sense alone. For example, templates used only in `!#insert` directives or base templates (you never call them alone, but their extended templates).

If you don't want to compile them into golang functions, add the `!#nocompile` directive.

## Invalid code

Some errors in your templates will be reported immediately:

    $ ftmpl example/invalid/
    invalid.tmpl    -> example/invalid/invalid.go
    go fmt example/invalid/invalid.go
    Cannot format example/invalid/invalid.go: exit status 1

    example/invalid/invalid.go:42:18: unknown escape sequence
           38:    result.WriteString(`    <body>
           39:    `)
           40:    //invalid.tmpl:         {{s "\ " }}
           41:    result.WriteString(fmt.Sprintf(`        %s
    -->    42:    `, _escape( "\ ")))
                                   ^
           43:    //invalid.tmpl:     </body>
           44:    result.WriteString(`    </body>
           45:    `)
           46:    //invalid.tmpl: </html>
           47:    result.WriteString(`</html>
    exit status 2

For others, you'll need to debug the Go code (every Go line is preceded with a Go comment with the original .tmpl file and tmpl code).

## Returning errors

Every template will result in *two* template functions. Both will execute the same code, but:

 * Use `func TMPLERRMyTemplate(args) (string, error)` if you expect your template to return errors, use this one. Use `!return "", err` to prematurely exit from the template with a proper error value.
 * With `func TMPLMyTemplate(args) string` if the template returns an error, the result is an empty string, and the error will be written in `os.Syserr`.

## Global declarations

In case you need a global declaration (outside of the template function), it can be done with `!#global`:

    !#global type Argument struct { Aaa string; Bbb int }
    !#arg arg Argument
    Aaa is {{s arg.Aaa }}, and Bbb is {{d arg.Bbb }}

The `Argument` struct will be declared outside the function body, and the function is now `TMPLMyTemplate(Argument)`.

## Special variables

`_template` is defined in the function and contains the current template name.

`_ftmpl` is a `bytes.Buffer` which contains the result of the template. It means that `{{! _ftmpl.WriteString("something") }}` is equivalent to `{{=s "something" }}`.

## Careful with Go code

Ftmpl allows you to write go code (instead of templating "metalanguages", like other templating engines), but be careful to not write too much of it. 
Ideally, the data should be prepared before the template function, and the only code in the template would be some basic formatting, a couple of `if`s and `for`loops.


# License

ftmpl is licensed under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0)
