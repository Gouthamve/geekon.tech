+++
draft = false
title = "Lexing and Parsing Inside the GoLang Compiler!"
date = "2016-09-10T19:55:25+05:30"

banner = "/img/posts/six/golang.png"

slug = "lexing-parsing-of-golang-compiler"
socialsharing = true

author = "Goutham"
authortwitter = "https://twitter.com/putadent"

categories = ["golang", "compiler"]
+++

# The GoLang Compiler

This was a Mini-assignment in my Compilers course at IIT Hyderabad to read existing implementations of lexers/parsers. I couldn't find any documentation on the Go Compiler hence, I decided to publish my report here. Please comment if you find any discrepancies.

## Some History
Go's compiler has a unique history. When the team first built Go, they used lex and bison for lexing and parsing. This was due to several reasons:

1. Go was unstable.
2. Go wasn't targeting compiler writers (Apparently, if you trying to bootstrap a language in the initial stages, the launguage will end up suitable for writing compilers! :P)

Now, after hitting 1.0, Go gave stability guarantees and was a great general purpose language. So the team decided to remove C from the compiler. Some motivations being:

1. Go is more fun than C :D
2. Need to know C to work on the Compiler. Only Go knowledge doesn't suffice.

While the reasons may sound silly, they are true. After moving from C to Go, there was huge surge in the contributors to the Go compiler, showing that C was a deterrent.

They built tools that automatically converted C to valid Go and then started cleaning up to make the code idiomatic Go.

## The Structure
Go is a simple language. The grammar and spec can be found [here](https://golang.org/ref/spec). Its so simple that the whole lexer is just 550 LOC. Go also has its parser written in Go, which amounts to 1800 LOC.

## The Lexer

The lexer itself is a golang package whose documentation and public API can be found [here](https://golang.org/pkg/go/scanner/). Like most hand-written lexers it is made up of a [huuuge switch case](https://github.com/golang/go/blob/master/src/go/scanner/scanner.go#L598-L761).


You initialise the scanner on a ```[]byte``` (which is all the text) and constantly call the ```Scan``` method. The Scanner struct has the current state and the methods defined on it are used to capture the tokens. For those tokens which take more than one look ahead, [these](https://github.com/golang/go/blob/master/src/go/scanner/scanner.go#L531-L565) functions are used to select tokens based on context.

## The Parser
One very frustrating aspect of the golang compiler is the lack of documentation. We have to go through the code to understand what is happening. Doing that for something as complex as a compiler ~~sucked~~ was tedious in the beginning but was acutally fun once I figured out whats happening.

So I am going to dump the code here as I wade through the parser, so if I don't clean it up, please be ready for a lot of code.
### Parser Struct

```
// The parser structure holds the parser's internal state.
type parser struct {
	file    *token.File
	errors  scanner.ErrorList
	scanner scanner.Scanner

	// Tracing/debugging
	mode   Mode // parsing mode
	trace  bool // == (mode & Trace != 0)
	indent int  // indentation used for tracing output

	// Comments
	comments    []*ast.CommentGroup
	leadComment *ast.CommentGroup // last lead comment
	lineComment *ast.CommentGroup // last line comment

	// Next token
	pos token.Pos   // token position
	tok token.Token // one token look-ahead
	lit string      // token literal

	// Error recovery
	// (used to limit the number of calls to syncXXX functions
	// w/o making scanning progress - avoids potential endless
	// loops across multiple parser functions during error recovery)
	syncPos token.Pos // last synchronization position
	syncCnt int       // number of calls to syncXXX without progress

	// Non-syntactic parser control
	exprLev int  // < 0: in control clause, >= 0: in expression
	inRhs   bool // if set, the parser is parsing a rhs expression

	// Ordinary identifier scopes
	pkgScope   *ast.Scope        // pkgScope.Outer == nil
	topScope   *ast.Scope        // top-most scope; may be pkgScope
	unresolved []*ast.Ident      // unresolved identifiers
	imports    []*ast.ImportSpec // list of imports

	// Label scopes
	// (maintained by open/close LabelScope)
	labelScope  *ast.Scope     // label scope for current function
	targetStack [][]*ast.Ident // stack of unresolved labels
}
```
So here the parser contains different fields, with the key fields being:

1. The file being parsed.
2. The list of Lexical Errors encountered until that point.
3. The scanner object.
4. The list of comments (With their positions).
5. The current Token
6. Lookahead Token (Only one?)
7. The literal for the current token
8. Some error recovery token/stuff? `syncPos`
9. Some non-syntactic parser control? `exprLev`
10. Scope info (packages, unresolved identifiers)
11. Additional scope info to support labels.


Now, the parser is `init`ed where it sets the base fields and consumes the lead/line token at the top. (This is usually the package licensing/documentation comment. [Example](https://golang.org/src/fmt/doc.go)).

Now, we parse a file via the [`ParseFile` function](https://github.com/golang/go/blob/master/src/go/parser/interface.go#L84-L124). This is just the API function which initializes the parser and calls the internal `parseFile` function. It parses:

SourceFile = [PackageClause](https://github.com/golang/go/blob/master/src/go/parser/parser.go#L2461-L2468) ";" { [ImportDecl](https://github.com/golang/go/blob/master/src/go/parser/parser.go#L2481-L2483) ";" } { [TopLevelDecl](https://github.com/golang/go/blob/master/src/go/parser/parser.go#L2487-L2489) ";" } .


One interesting thing is that the semi-colons are optional for the programmer and are automatically inserted by the lexer. 

---
### Some sample parsing code.
Parsing declarations is a fairly straight forward task as each declaration has a keyword to identify the type of declaration.

* [**Function Declaration**](https://github.com/golang/go/blob/master/src/go/parser/parser.go#L2369-L2417): 


```
FunctionDecl = "func" FunctionName ( Function | Signature ) .
FunctionName = identifier .
Function     = Signature FunctionBody .
FunctionBody = Block .
```
We simply look for the _func_ keyword and then an _identifier_ and then a _Signature_ (without the body) or the _Function_ definition.
Now to differentiate between a _Signature_ and a _Function_, we lookahead for the **LBRACE** token.

* **Parameter Declaration**:

```
ParameterList  = ParameterDecl { "," ParameterDecl } .
ParameterDecl  = [ IdentifierList ] [ "..." ] Type .
```

The following is valid go:
```
func()
func(x int) int
func(a, _ int, z float32) bool
func(a, b int, z float32) (bool)
func(prefix string, values ...int)
func(a, b int, z float64, opt ...interface{}) (success bool)
func(int, int, float64) (float64, *[]int)
func(n int) func(p *T)
```

One interesting problem here is that parameter parsing requires arbitrary lookahead as we can do the following where we dont know if _string_ refers to a variable named string or a type string:

```
func(string, a, b, c, d, e int, z float32) (bool)
```

This is solved by imposing the following restriction:

>
Within a list of parameters or results, the names (IdentifierList) must either all be present or all be absent. If present, each name stands for one item (parameter or result) of the specified type and all non-blank names in the signature must be unique. If absent, each type stands for one item of that type. Parameter and result lists are always parenthesized except that if there is exactly one unnamed result it may be written as an unparenthesized type. 


Parsing this via handwritten parser is simple, else, if it was a generated parser, we would need to do a pass over the AST to sort this out.

* **Type Declaration**:

```
Type      = TypeName | TypeLit | "(" Type ")" .
TypeName  = identifier | QualifiedIdent .
TypeLit   = ArrayType | StructType | PointerType | FunctionType | InterfaceType | SliceType | MapType | ChannelType .
```

Even here, we simply look at one token and then move into parsing that:

```
func (p *parser) tryIdentOrType() ast.Expr {
	switch p.tok {
	case token.IDENT:
		return p.parseTypeName()
	case token.LBRACK:
		return p.parseArrayType()
	case token.STRUCT:
		return p.parseStructType()
	case token.MUL:
		return p.parsePointerType()
	case token.FUNC:
		typ, _ := p.parseFuncType()
		return typ
	case token.INTERFACE:
		return p.parseInterfaceType()
	case token.MAP:
		return p.parseMapType()
	case token.CHAN, token.ARROW:
		return p.parseChanType()
	case token.LPAREN:
		lparen := p.pos
		p.next()
		typ := p.parseType()
		rparen := p.expect(token.RPAREN)
		return &ast.ParenExpr{Lparen: lparen, X: typ, Rparen: rparen}
	}

	// no type found
	return nil
}
```

### Parsing Statements

```
Statement =
	Declaration | LabeledStmt | SimpleStmt |
	GoStmt | ReturnStmt | BreakStmt | ContinueStmt | GotoStmt |
	FallthroughStmt | Block | IfStmt | SwitchStmt | SelectStmt | ForStmt 
	| DeferStmt .

SimpleStmt = EmptyStmt | ExpressionStmt | SendStmt | IncDecStmt | Assignment 
        | ShortVarDecl .
```
Code:
```
	switch p.tok {
	case token.CONST, token.TYPE, token.VAR:
		s = &ast.DeclStmt{Decl: p.parseDecl(syncStmt)}
	case
		// tokens that may start an expression
		token.IDENT, token.INT, token.FLOAT, token.IMAG, token.CHAR, token.STRING, token.FUNC, token.LPAREN, // operands
		token.LBRACK, token.STRUCT, token.MAP, token.CHAN, token.INTERFACE, // composite types
		token.ADD, token.SUB, token.MUL, token.AND, token.XOR, token.ARROW, token.NOT: // unary operators
		s, _ = p.parseSimpleStmt(labelOk)
		// because of the required look-ahead, labeled statements are
		// parsed by parseSimpleStmt - don't expect a semicolon after
		// them
		if _, isLabeledStmt := s.(*ast.LabeledStmt); !isLabeledStmt {
			p.expectSemi()
		}
	case token.GO:
		s = p.parseGoStmt()
	case token.DEFER:
		s = p.parseDeferStmt()
	case token.RETURN:
		s = p.parseReturnStmt()
	case token.BREAK, token.CONTINUE, token.GOTO, token.FALLTHROUGH:
		s = p.parseBranchStmt(p.tok)
	case token.LBRACE:
		s = p.parseBlockStmt()
		p.expectSemi()
	case token.IF:
		s = p.parseIfStmt()
	case token.SWITCH:
		s = p.parseSwitchStmt()
	case token.SELECT:
		s = p.parseSelectStmt()
	case token.FOR:
		s = p.parseForStmt()
	case token.SEMICOLON:
		// Is it ever possible to have an implicit semicolon
		// producing an empty statement in a valid program?
		// (handle correctly anyway)
		s = &ast.EmptyStmt{Semicolon: p.pos, Implicit: p.lit == "\n"}
		p.next()
	case token.RBRACE:
		// a semicolon may be omitted before a closing "}"
		s = &ast.EmptyStmt{Semicolon: p.pos, Implicit: true}
	default:
		// no statement found
		pos := p.pos
		p.errorExpected(pos, "statement")
		syncStmt(p)
		s = &ast.BadStmt{From: pos, To: p.pos}
	}

	return
}
```

This is again a one token lookahead parsing.


### Parsing Conculsion
This is an LL type parsing with LL(1) for the most part. But sometimes, like the parameter scenario, we would need to do extra lookahead to sort this out.

Overall the code is clean and easy to follow. <3 GoLang.


## Rest of the Toolchain
Go has a new SSA backend that shipped very recently (v1.7 came last month! Read more [here](https://blog.golang.org/go1.7)). Some docs can be found here: https://godoc.org/golang.org/x/tools/go/ssa

And backend specific code can be found in `src/cmd/compile/internal`. 

_PS: I am pretty sure the assignment's motives were fulfilled!_