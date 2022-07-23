## Front-End SemGuS Language

The SemGuS front-end language format is based on SMT-LIB2 and inspired by SyGuS. It allows specifying
the grammar of a synthesis problem, semantics for each production, and constraints on the desired program.

[The SemGus Front-End Language \[pdf\]](res/semgus-lang.pdf)

### Example: `max2` (Expressions)

The following is an example synthesis problem for finding a program that returns the maximum of its
two arguments. The grammar consists of the following syntax constraints:

```
E ::= x | y | 0 | 1 | E + E | if B then E else E
B ::= true | false | !B | B & B | E < E
```

Additionally, it contains the semantics associated with each production. The resultant specification
looks as follows:

```lisp
;; Metadata about the synthesis problem
(set-info :author "Jinwoo Kim")
(set-info :realizable true)

;; Declarations of term types and their syntactic constructors
;; Terms of type E will be integer-valued expressions; terms of type B will be Boolean-valued expressions 
(declare-term-types
  ((E 0) (B 0))
  
   ;; Syntactic constructors for integer expressions
  ((($x)          ; E -> x
    ($y)          ;    | y
    ($0)          ;    | 0
    ($1)          ;    | 1
    ($+ E E)      ;    | (+ E E)
    ($ite B E E)) ;    | (ite B E E)
   
   ;; Syntactic constructors for Boolean expressions
   (($true)     ; B -> true
    ($false)    ;    | false
    ($not B)    ;    | (not B)
    ($and B B)  ;    | (and B B)
    ($< E E)))) ;    | (< E E)

;; Declarations of semantics associated with the syntax defined above
(define-funs-rec
  ;; Declare the signatures of semantic relations for each term type
  ((E.Sem ((t_e E) (x Int) (y Int) (r Int)) Bool)    ; Integer exprs relate inputs (x: Int, y: Int) to outputs r: Int
   (B.Sem ((t_b B) (x Int) (y Int) (rb Bool)) Bool)) ; Boolean exprs relate inputs (x: Int, y: Int) to outputs rb: Bool

   ;; Declare the semantic rules for E
  ((! (match t_e
         ;; Leaves, with a CHC body (= r _)
        (($x (= r x))
         ($y (= r y))
         ($0 (= r 0))
         ($1 (= r 1))

         ;; + operator, with other E as children
         (($+ et1 et2)
           ;; We quantify over the intermediate variables in the CHC
           (exists ((r1 Int) (r2 Int))
             (and
               ;; The CHC body contains the semantic relations for child terms
               (E.Sem et1 x y r1)
               (E.Sem et2 x y r2)
               ;; ...as well as the CHC constraint
               (= r (+ r1 r2)))))
         
         ;; If-then-else, with an additional B as a child
         (($ite bt1 et1 et2)
           ;; Note there are two CHCs bodies for this one
           ;; One when B is true...
           (exists ((rb Bool))
             (and
               (B.Sem bt1 x y rb)
               (= rb true)
               (E.Sem et1 x y r)))
           ;; ...and one when B is false
           (exists ((rb Bool))
             (and
               (B.Sem bt1 x y rb)
               (= rb false)
               (E.Sem et2 x y r))))))
     
     ;; Mark x and y as "inputs" and r as an "output" for the E.Sem relation
     :input (x y) :output (r))
   
   ;; Declare the semantic rules for B
   (! (match t_b
        (($true (= rb true))
         ($false (= rb false))
         (($not bt1)
           (exists ((rb1 Bool))
             (and
               (B.Sem bt1 x y rb1)
               (= rb (not rb1)))))
         (($and bt1 bt2)
           (exists ((rb1 Bool) (rb2 Bool))
             (and
               (B.Sem bt1 x y rb1)
               (B.Sem bt2 x y rb2)
               (= rb (and rb1 rb2)))))
         (($< et1 et2)
           (exists ((r1 Int) (r2 Int))
             (and
               (E.Sem et1 x y r1)
               (E.Sem et2 x y r2)
               (= rb (< r1 r2)))))))
     :input (x y) :output (rb))))

;; Declare the synthesis objective
;; We want an integer expression called "max2"
(synth-fun max2 () E)

;; We provide an example-guided constraint
(constraint (E.Sem max2 4 2 4))
(constraint (E.Sem max2 2 5 5))

;; Synthesize "max2"!
(check-synth)
```

We now examine some of these pieces in more detail.

#### Metadata

SemGuS problems can have optional metadata about the problem itself:

```lisp
(set-info :author "Jinwoo Kim")
(set-info :realizable true)
```

Solvers must not use this metadata when solving the synthesis problem; it is intended to be used
for automated benchmark suites and other testing tools.

#### Term Type Declarations

We use the `declare-term-types` command to define data types for terms in the target grammar, as well as
the syntactic constructors used to construct them. Each declaration is additionally given an arity as in
the SMT-LIB sort declaration syntax, which should be zero for term types.

```lisp
(declare-term-types ((E 0) (B 0) ...) ((<constructors for E>) (<constructors for B>) ...))
```

Note that the lists of syntactic constructor declarations should be in the same order as the respective
term type declarations.

The syntactic constructors associated with a term type define the possible ways to construct a term of
that type. Each constructor consists of a name, optionally followed by zero or more term types for any
child terms.

```lisp
(($x)         ; Leaf term with no children
 ($+ E E)     ; Operator with two children of term type E
 ($ite B E E) ; Operator with three children of types B, E, E
 ...) 
```

#### Semantics Declarations

To associate semantic rules with a previously-declared syntax, we use the `define-funs-rec` command,
which lets us declare semantic relations in the form of functions mapping values that the relations
range over to Booleans. For example, the `E.Sem` relation expresses that the term `t_e: E` evaluates to
`r` in a context given by `x` and `y`, and so we represent it as a function `(t_e, x, y, r) -> Bool`.

```lisp
(define-funs-rec
  ((E.Sem ((t_e E) (x Int) (y Int) (r Int)) Bool)   ; Signature of sem. relation for E
   (B.Sem ((t_b B) (x Int) (y Int) (rb Bool)) Bool) ; Signature of sem. relation for B
   ...)
  
  (<E.Sem body> <B.Sem body> ...))
```

Note that the bodies of the semantic relations should appear in the same order that their respective
relations are defined in.

Each semantic relation body should consist of a `match`-form that matches on the syntactic constructors
for the term type. This way, we can assign to each constructor (corresponding to productions in the
target grammar) a separate set of CHCs as semantic rules. The body of each match case corresponds to a
single CHC. For terms with multiple semantic rules (for example, if-then-else conditionals or while
loops), we may declare multiple bodies for a single match case.

```lisp
(match t_e
  (($x <x semantic rule>)           ; Leaf term with no children
   (($+ et1 et2) <+ semantic rule>) ; Operator with two children et1: E, et2: E
   (($ite bt1 et1 et2)              ; Operator with three children and two semantic rules
     <if-then-else semantic rule one>
     <if-then-else semantic rule two>))
  ...)
```

#### CHC Declarations for Semantic Rules

The semantic rules themselves are SMT-LIB terms expressing a relation between the arguments to the
function as declared above. They should consist of an `exists`, which allows for the declaration of
intermediate variables in the CHC, whose inner term is an `and`, whose clauses are either semantic
relations for child terms or conjuncts for the CHC's constraint.

```lisp
;; CHC for the + operator with children et1: E and et2: E
;; forall r1, r2. E.Sem(et1, x, y, r1) /\ E.Sem(et2, x, y, r2) /\ r = r1 + r2
;;             => E.Sem(($+ et1, et2), x, y, r)
(exists ((r1 Int) (r2 Int)) ; Declare intermediate integer variables r1 and r2
  (and
    (E.Sem et1 x y r1) ; Semantic relation for the first child term et1
    (E.Sem et2 x y r2) ; Semantic relation for the second child term et2
    (= r (+ r1 r2))))  ; Constraint: the result of + is the sum of the results of et1 and et2
```

If no intermediate variables are needed, the `exists` may be omitted. Moreover, if there is only one
clause in the `and` (often the case for leaf terms, where there are no child terms to specify relations
for), it may be omitted.

```lisp
;; CHC for the x leaf term, which has no children
;; r = x => E.Sem($x, x, y, r)
(= r x) ; Constraint: the result is the input x
```

#### Input and Output Annotations

It is possible to annotate an argument to a semantic relation as an "input" or an "output". In semantics
where program evaluation may be regarded as a transition system, such an annotation asserts that an
argument represents pre-condition or post-condition data, respectively, for a transition corresponding to
the term. For example, in an imperative semantics, the pre-condition data are the states of the variables
prior to a statement's execution, and the post-condition data are the states afterwards.

To annotate the arguments of a semantic relation, we wrap its body declaration with a `!` and append the
annotations behind it.

```lisp
;; For the semantic relation E.Sem(t_e, x, y, r), we annotate x and y as inputs and r as an output
(! (match t_e (<match cases>)) :input (x y) :output (r))
```

Note that these annotations are strictly optional, and only serve to assist certain solvers that may be
able to make use of them.

#### Synthesis Objective Specification

The `synth-fun` command gives a name to a term to be synthesized and specifies its term type. This is
the equivalent of the `synth-fun` command in the SyGuS specification language.

```lisp
(synth-fun max2 () E) ; Synthesize a term max2 of term type E
```

Constraints are specified as predicates over the semantic relations of the grammar. Multiple constraints
can be asserted by multiple invocations of the `constraint` command. By asserting that `E.Sem` holds for
`max2` on some sets of constants, we may provide an example-guided constraint for it.

```lisp
(constraint (E.Sem max2 4 2 4)) ; (max2 4 2) ~> 4
(constraint (E.Sem max2 2 5 5)) ; (max2 2 5) ~> 5
```

It is also possible to define a logical constraint by providing an arbitrary SMT-LIB term.

```lisp
;; forall x, y, r. (max2 x y) = r <=> (x = r \/ y = r) /\ r >= x /\ r >= y
(constraint (forall ((x Int) (y Int) (r Int))
              (= (E.Sem max2 x y r)
                 (and (or (= x r) (= y r))
                      (>= r x)
                      (>= r y)))))
```

By defining `max2` as the synthesis objective in the `synth-fun` command, we mark it as a single term
that must satisfy all constraints we give it. Finally, once we've declared our synthesis objective and
all the constraints on it, we can use the `check-synth` command to ask the solver to solve the synthesis
problem.

```lisp
(check-synth) ; You've got this!
```

### Example: `max2` (Imperative)

The preceding grammar uses only integer and Boolean expressions, and an equivalent problem could be
written in SyGuS, assuming standard semantics. However, SemGuS is not limited to only grammars of
expressions; an alternate, imperative grammar is also possible, over the following syntax:

```
S ::= x = E | y = E | c = E | if B then E else E
E ::= x | y | 0 | 1
B ::= true | false | not B | B & B | E < E
```

This time, imperative semantics are specified for the `Start` non-terminal's productions.

```lisp
(set-info :author "Jinwoo Kim")
(set-info :realizable true)

(declare-term-types
  ((S 0) (E 0) (B 0)) ; S: statements, E: integer expressions, B: Boolean expressions
  
   ;; Statements have assigment operators for x and y, as well as an auxiliary variable c
   ;; Additionally, we have conditionals and sequencing (the "semicolon operator")
  ((($x= E)     ; S -> x := E
    ($y= E)     ;    | y := E
    ($c= E)     ;    | c := E
    ($if B S S) ;    | if B then S else S
    ($seq S S)) ;    | S ; S
   
   (($x)           ; E -> x
    ($y)           ;    | y
    ($c)           ;    | c
    ($0)           ;    | 0
    ($1)           ;    | 1
    ($+ E E)       ;    | (+ E E)
    ($ite B E E )) ;    | (ite B E E)
   
   (($true)     ; B -> true
    ($false)    ;    | false
    ($not B)    ;    | (not B)
    ($and B B)  ;    | (and B B)
    ($< E E)))) ;    | (< E E)

(define-funs-rec
  ;; Same x and y as before, but with an auxiliary variable c.
  ;; Start is over statements, with x, y, c the input state and rx, ry, rc the output state
  ((S.Sem ((t_s S) (x Int) (y Int) (c Int) (rx Int) (ry Int) (rc Int)) Bool)
   (E.Sem ((t_e E) (x Int) (y Int) (c Int) (r Int)) Bool)
   (B.Sem ((t_b B) (x Int) (y Int) (c Int) (rb Bool)) Bool))

  ;; Semantics for statements
  ((! (match t_s
        ((($x= et1)
           (exists ((v Int))
             (and
               (E.Sem et1 x y c v)
               (= rx v) ; x gets set to the result...
               (= ry y) ; ...but y and c are unchanged
               (= rc c))))
         (($y= et1)
           (exists ((v Int))
             (and
               (E.Sem et1 x y c v)
               (= rx x)
               (= ry v)
               (= rc c))))
         (($c= et1)
           (exists ((v Int))
             (and
               (E.Sem et1 x y c v)
               (= rx x)
               (= ry y)
               (= rc v))))
         (($if tb s1 s2)
           (exists ((vb Bool) (rx1 Int) (ry1 Int) (rc1 Int) (rx2 Int) (ry2 Int) (rc2 Int))
             (and
               (B.Sem tb x y c vb)          ; Evaluate the condition to vb...
               (S.Sem s1 x y c rx1 ry1 rc1) ; ...and take both branches...
               (S.Sem s2 x y c rx2 ry2 rc2)
               (= rx (ite vb rx1 rx2))      ; ...then choose from the two results based vb
               (= ry (ite vb ry1 ry2))
               (= rc (ite vb rc1 rc2)))))
         (($seq s1 s2)
           (exists ((ix Int) (iy Int) (ic Int))
             (and
               (S.Sem s1 x y c ix iy ic)         ; Go through the first statement...
               (S.Sem s2 ix iy ic rx ry rc)))))) ; ...then feed its results into the second
     :input (x y c) :output (rx ry rc))
   
   ;; The rest of the grammar is essentially the same as the expression-based grammar above
   (! (match t_e
        (($x (= r x))
         ($y (= r y))
         ($c (= r c))
         ($0 (= r 0))
         ($1 (= r 1))
         (($+ et1 et2)
           (exists ((r1 Int) (r2 Int))
             (and
               (E.Sem et1 x y c r1)
               (E.Sem et2 x y c r2)
               (= r (+ r1 r2)))))
         (($ite bt1 et1 et2)
           (exists ((rb Bool))
             (and
               (B.Sem bt1 x y c rb)
               (= rb true)
               (E.Sem et1 x y c r)))
           (exists ((rb Bool))
             (and
               (B.Sem bt1 x y c rb)
               (= rb false)
               (E.Sem et2 x y c r))))))
     :input (x y c) :output (r))
   (! (match t_b
        (($true (= rb true))
         ($false (= rb false))
         (($not bt1)
           (exists ((rb1 Bool))
             (and
               (B.Sem bt1 x y c rb1)
               (= rb (not rb1)))))
         (($and bt1 bt2)
           (exists ((rb1 Bool) (rb2 Bool))
             (and
               (B.Sem bt1 x y c rb1)
               (B.Sem bt2 x y c rb2)
               (= rb (and rb1 rb2)))))
         (($< et1 et2)
           (exists ((r1 Int) (r2 Int))
             (and
               (E.Sem et1 x y c r1)
               (E.Sem et2 x y c r2)
               (= rb (< r1 r2)))))))
     :input (x y c) :output (rb))))

;; Give me a statement called "max2i"
(synth-fun max2i () S)

;; As we don't care about the final value of c, we allow it to be anything by quantification
;; We arbitrarily set its initial value to zero
(constraint (exists ((c Int)) ; (x=4, y=-1) ~> (x=4, y=-1)
              (S.Sem max2i 4 (- 1) 0 4 (- 1) c)))
(constraint (exists ((c Int)) ; (x=-2, y=3) ~> (x=3, y=-2)
              (S.Sem max2i (- 2) 3 0 3 (- 2) c)))

;; Synthesize "max2i"!
(check-synth)
```
