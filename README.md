# PEG parser generator

This readme aims at giving a detailed look on how this peg parser works, assuming that the reader already knows about PEG grammars and general parsing.

## nT_pred_order.pvs
*Technical definitions used for predicate manipulations*

Here are defined a boolean triple `nTinst` (short for *non-terminal properties instance*) to represent the grammar properties (can fail, can succeed without consuming something, can succeed consuming something) of a given grammar node. To store the properties of all the non-terminal interpretations, we use the `nTprop` object. We define an order on both, and the usual properties : transitivity and reflexivity, plus a distributivity of the order on `nTprop` over instances of `nTinst`.

## array_sum.pvs
*Technical definitions used for counting the number of times a predicate is satisfied in a `nTprop` object*

The `aux` function is just a tail-recursive function that iterates over the array. Two main results are shown : the aux function is growing (for the nonTerminal ordering) and injective.

## delta.pvs
*Main file for peg grammar definition, properties computation and wellformedness*

### `peg THEORY`
This theory is parameterized by the type of the terminals, their ordering, and the number of nonterminals
The `peg` datatype is made of constructors that directly represent the possible patterns of a peg grammar.
The `pegMeasure` is a simple `reduce_nat`. Three results are shown : the measure is growing and injective regarding the *subterm* relationship, and the *subterm* relationship is transitive.

### `wf_peg THEORY`
This theory is parameterized by the same parameters as `peg`.
This theory aims at expressing what a wellformed grammar is, based on properties calculation.
The `grammar_props` function recursively computes the properties of a grammar node based on the *already-known* properties of the non-terminals (given as an argument). Then we can compute all accessible new properties, adding them as we go over all the non terminals with the `recompute_nonTerminals_properties` function. The fix point of accessible properties is obtained with the `compute_properties` function, that calls the previous one over and over again until no new properties are found. The obtained set of properties, of type `fix_point` is *coherent* (not contradicted by iteself) and *maximal* (recomputing properties based on it cannot lead to new ones).

Two types of wellformedness are defined :

* **complete** : a completely-wellformed grammar node is a node where no subterm is of the form `star(e)` or `plus(e)` with `e` being able to succeed without consuming any input (thus looping forever). To say it in other words : *a completely-wellformed grammar does not have structural loops*
* **strong** : a strongly-wellformed grammar node (relative to a non terminal A) is a node that only uses non terminals that are strictly less than A, unless a sequence is found, with the first argument not being able to succeed without consuming any input. Then the check switches to the complete wellformedness. To say it in other words : *a strongly wellformed grammar is allowed to use higher non terminals only when we are sure that it consumed at least one character*.

The `grammar_wellformedness` function works for both kinds by switching the last argument. A *wellformed set of non terminals* (`WF_nT`) is defined.

## pre_ast.pvs
*Definition of an abstract syntax tree (ast) structure*
The datatype `pre_ast` aims at capturing all parsing paths possible. A few structural conditions are already included in the constructors (for example, the leftmost branch correspond to a path that the parser has to explore, and thus cannot contain any skip node).

## ast.pvs
*Main file for ast properties : definition of a meaningful and a wellformed ast*

The `astType` captures the fact that a tree is either a coherent proof of success or failure, or is incoherent (undefined). The check is done by the recursive `astType?` function. A `astMeaningful` tree is a tree that is either of type `success` or `failure`.

The `astWellformed?` function recursively checks that the tree correspond to a valid proof of parse. Two results are shown : a wellformed tree is also meaningful, and a wellformed tree starting at `s` and ending at `e` as to verify `s <= e`.

## lex3.pvs
*Technical definition of triple lexical ordering*

## lex4.pvs
*Technical definition of quadruple lexical ordering*

## parser.pvs
*Main file for the reference parser generator definition*

The argument of the parser are the following :
* `P_exp` : the set of non terminal interpretations, of type `WF_nT` (strongly wellformed)

* `A` : the current non terminal
* `G` : the current grammar node (as to be a subterm of `P_exp(A)`)
* `inp`: the input array
* `b` : the bound of the input
* `s` : the current index
* `s_T` : the index when the parse of the current nonTerminal started

The output type carries a lot of information. A returned tree T verifies :

* `s(T) = s` : it starts at the current index
* `e(T) <= b` : it ends before or at the bound
* if G was a star node, then T is too
* if G was a plus node, then T is too
* T is not a skip (a skip does not represent a parsing result)
* if T is a success, and did not consume anything, it implies that G had the property `P_0`
* if T is a success, and did consume something, it implies that G had the property `P_>0`
* if T is a failure, it implies that G had the property `P_f`

The parsing termination is proved using a quadruple lexicographic order :

 * the parser goes down the grammar (`pegMeasure(G)`), unless it finds a star, a plus or a nonTerminal operator
 * if a star or a plus is found, then the grammar node stays constant, but at least one character must be consumed and thus, `s` increases.
 * if a nonTerminal node is found, `s` does not change, and the new grammar node might be anything. But, as `P_exp` is strongly wellformed, we either have a non terminal strictly lower than the current one, or at least one character was consumed, so `s` is greater than `s_T` (so the recursive call with `s_T` set as `s` is legit)

Most of the proof work is done in the tccs.

## packrat_parser.pvs
*Definition of a parser generator that uses memoization*
  The packrat_parser theory has a similar parser with a result that is a table mapping each start index and nonterminal to an AST that matches the one returned by the reference parser parsing, and an updated table.
