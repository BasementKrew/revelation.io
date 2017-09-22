---
layout: post
title:  "Gopher Go! - Builtin"
date:   2014-06-30 08:00:00
categories: 'austin'
summary: "Our journey through the documentation now lands us on the Builtin package. In today's article we will explore how the Builtin in package works."
tags: Go
keywords: Go golang packages pkg builtin c compiler
---

Ah, builtin in types and functions. They are what makes data computations go around in most languages and golang is no different. The builtin package is not so much a "package" as it is documentation for Go's different identifiers. You may have noticed this by the fact you don't have to import the builtin package to have access to all the types and functions list in the docs. Since there is much to cover in this package, let's talk a bit about how golang exactly implements these builtin types. For those who have been through the golang source, you will notice that the builtin package directory only has the one go file for documentation. If you are curious like me, you might be wondering where these types and functions are hiding out in the golang source. In a very anticlimactic turn of events, it was "builtin" (who would have figured?) to the golang compiler, `gc`. Alright, time for my usual disclaimer here. I tend to think of this meme when I get ready to science:

![](http://thumbpress.com/wp-content/uploads/2013/05/I-Have-No-Idea-What-Im-Doing-1.jpg)

I'm by no means a compiler expert (although I would like to be!) so don't take my finding as gospel. Without that out of the way, let's dig in. If you peak at the source and take a look at cmd instead of the package directory, we notice a bunch of seemingly randomly named folder along with some familiar ones such as `cgo`, `go`, `gofmt` and `gc`. I tend to like to look for docs when I get started and long behold, pretty much every directory had one. The seemingly random named folders are actually the linkers for the different OS architectures. The rest of the directories are array of tools that we won't get into right now. If we turn our attention back to `gc`, we notice that `gc` is in fact a modified version of the Plan 9 compiler. When we open up the referenced PDFs, we notice the authors include Rob Pike and Ken Thompson, both core contributors to golang, hence the use of a modified Plan 9 compiler. Both PDFs are pretty high level and about the original C compiler, but are interesting reads none the less. Normally my next would be to just to read through the source, but since there is quite a bit, I figured I would turn to the Internet to see if I could find any pointers. Always wanting to give credit where credit is due, I found a stackoverflow post that had just what I was looking for. The link to it is at the end of the article. Instead of a `make` like the SO post, let's take a look at the `append` function.

```cpp

  case OAPPEND:
    ok |= Erv;
    args = n->list;
    if(args == nil) {
      yyerror("missing arguments to append");
      goto error;
    }

    if(count(args) == 1 && !n->isddd)
      typecheck(&args->n, Erv | Efnstruct);
    else
      typechecklist(args, Erv);

    if((t = args->n->type) == T)
      goto error;

    // Unpack multiple-return result before type-checking.
    if(istype(t, TSTRUCT)) {
      t = t->type;
      if(istype(t, TFIELD))
        t = t->type;
    }

    n->type = t;
    if(!isslice(t)) {
      if(isconst(args->n, CTNIL)) {
        yyerror("first argument to append must be typed slice; have untyped nil", t);
        goto error;
      }
      yyerror("first argument to append must be slice; have %lT", t);
      goto error;
    }

    if(n->isddd) {
      if(args->next == nil) {
        yyerror("cannot use ... on first argument to append");
        goto error;
      }
      if(args->next->next != nil) {
        yyerror("too many arguments to append");
        goto error;
      }
      if(istype(t->type, TUINT8) && istype(args->next->n->type, TSTRING)) {
        defaultlit(&args->next->n, types[TSTRING]);
        goto ret;
      }
      args->next->n = assignconv(args->next->n, t->orig, "append");
      goto ret;
    }
    for(args=args->next; args != nil; args=args->next) {
      if(args->n->type == T)
        continue;
      args->n = assignconv(args->n, t->type, "append");
    }
    goto ret;
```

The above code is a chunk from `gc` `typecheck.c`. This simply does some type checking and does the symbol substitution of the Go `append` method into the OAPPEND symbol for `gc`. Once the symbol replacement is replaced and gets through the type checking, the appropriate runtime function is called for the OAPPEND symbol. Next we take a look at `walk.c` in `gc`.

```cpp

  case OAPPEND:
    if(n->isddd)
      n = appendslice(n, init); // also works for append(slice, string).
    else
      n = append(n, init);
    goto ret;
```

Here we see the `appendslice` or `append` function being called. Instead of digging into the `appendslice` or `append` (There a ton of C source that would take forever to get through and this is a golang series!) we can look at the comment to see what these function "expand" into. Let's do `appendslice`:

```cpp
// expand append(l1, l2...) to
//   init {
//     s := l1
//     if n := len(l1) + len(l2) - cap(s); n > 0 {
//       s = growslice(s, n)
//     }
//     s = s[:len(l1)+len(l2)]
//     memmove(&s[len(l1)], &l2[0], len(l2)*sizeof(T))
//   }
//   s
//
// l2 is allowed to be a string.
```

Then `append`:

```cpp
// expand append(src, a [, b]* ) to
//
//   init {
//     s := src
//     const argc = len(args) - 1
//     if cap(s) - len(s) < argc {
//      s = growslice(s, argc)
//     }
//     n := len(s)
//     s = s[:n+argc]
//     s[n] = a
//     s[n+1] = b
//     ...
//   }
//   s
```

Not too bad. Overall compiler design and features is endless vortex of awesome computer science nerdiness that I could get lost in for hours, but for both our sakes I will wrap it up here. The end take away here is that these types and functions are actually implemented in the actual golang compiler, hence there documentation being perfectly named package of "builtin". As always, any questions, comments, and additions are welcomed.

[SO Post](http://stackoverflow.com/questions/18512781/built-in-source-code-location)

[Twitter](https://twitter.com/acmacalister)