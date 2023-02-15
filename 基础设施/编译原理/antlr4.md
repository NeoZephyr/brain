```g4
grammar Hi;
r : 'hello' ID ;
ID : [a-z]+ ;
WS : [ \t\r\n]+ -> skip ;
```

lexer 规则名称用大写开头，parser 规则名称用小写开头

## 常用结构

### Sequence
```g4
(INT)+
INT+

// sequence with a ',' separator
row : field (',' field)* ;

('=' expr)?
('=' expr | )
```

### Choice
```g4
field : INT | STRING ;
```

### Token Dependency
```g4
vector : '[' INT+ ']' ;
```

### Nested Phrase
```g4
expr: ID '[' expr ']' // a[1], a[b[1]], a[(2*b[1])]
	  | '(' expr ')' // (1), (a[1]), (((1))), (2*a[1])
		| INT // 1, 94117
		;

expr : expr '^'<assoc=right> expr // ^ operator is right associative
		 | expr '*' expr // match subexpressions joined with '*' operator
		 | expr '+' expr // match subexpressions joined with '+' operator
		 | INT // matches simple integer atom
		 ;
```

## 编译
```bash
antlr4 Hi.g4
javac *.java

grun Hi r -tokens
grun Hi r -tree -gui
```
