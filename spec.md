# **Preliminary Spec**

I consider this document a first attempt at defining the structure of the coding portion
of the GISP.

## **A Theoretical Aside**

Just making sure I understand the theoretical model correctly. Please correct any incorrect interpretation below.

The prose object model is a way of representing a legal document as a node on a graph (in the mathematical sense of the word). The document is defined by other nodes it references on a graph (this abstraction is described in Gabriel’s Monads, and is the abstraction that inspired PageRank as well). The cool thing about the prefix system (and the instantiation construct) is that *they provide a stronger differentiation of a node’s worldview*. Not only is a node’s view affected by its location in the graph (aka my perspective is shaped by location on earth. If I’m in Paris, I see the Eiffel Tower, if I’m in NYC, I see the Empire State), but also that prefixing provides a *different ontology*. Not only does the node have a unique view into the world – it has its *own unique world* that only it inhabits. A node’s world is defined by its **location in the graph** (its edges) and **its prefixes**. Let me continue the geographic analogy, except in this case I will add the concept of prefixing back in. Consider again two nodes – one Parisian, one New Yorker. Not only are their perspectives different spatially (one sees Paris, one sees New York), but they literally live in different worlds. The Parisian operates with a certain set of prefixes (French, Parisian, etc) that defines how its perspective. Consider the object my_city. Both reference the file “city”, but one sees Parisian.city and the other sees New_Yorker.city. Same object, different instantiation. They both reference the my_city node, but instantiate it differently. Same graph, same platonic objects, but different names, different instantiations.

With that philosophical aside over with, I will now describe how I believe we should architect the program. Note that I have not yet looked at the Perl implementation. I don't know the language and also appreciate a fresh start - I'd be happy to refer to it later if you guys believe it would be helpful. I will begin by attempting to implement an abstract version of the model (No I/O, no concrete implementation, just pure data strctures and functions operating on those data structures).

## **The Spec**

### **Defining the Structure**

I propose that we begin by defining a set of function contracts that defines a Prose Object Model in the abstract. Each node shall be, in this document, hereafter referred to as a **Document** (in the spirit of the contracts as artificats doc you guys linked). Each Model contains a value (its text content which may be sprinkled with keys, saved as a list of Tokens (either literal or a reference)), an edge list, and an edge lookup (where keys are linked to edges). Note that the value is not actually seperate, but are an ordered list of *edges to terminal nodes of the graph*. Terminal nodes are nodes that have no outgoing edges, and are the only nodes in the graph with value (a string, a number, etc). Non terminal nodes will only contain a list of keys and a list of edges. Our first implementation should consider keys in the abstract (as a *symbolic link to some Document*, not a string enclosed in braces), and values in the same way (not as a string or filepath etc, but a *pointer to another Document*). In this case, the node that contains “my age is {jake.name}” will actually contain (in its edge list, in addition to other references) a reference to a node that contains “my age is “ and a symbolic link that contains {jake.name} (if this reference to this link cannot be found, it can be considered a reference to a string ‘{jake.name}’). Note that I have not yet thought out how we will represent the difference between a link determined by a key (which involves prefixing) vs a link w/ no key. It is possible that a keyed link can just be considered a subclass of Edge.

A non-terminal document only holds and an identifier, tokens (terminal and hanging), and links. It holds the tokens in one basket, and the links in another - a value basket and a map basket respectively. The value basket is the body of the document, which is to be rendered in sequential order. When rendering a terminal token, you can load the value of the terminal document directly. When loading a hanging token, a search must be exectuted. This is where the map comes into play.

The map basket holds, in its values, all of the other documents that document knows about. These are represented as links. The keys of the map are bound tokens. The values of the map may be conceptualized as roads, whereas the keys of the map are road signs.

When a hanging token is referenced, a search commences. It ends when it finds a matching bound token or exhausts the available subgraph and knows that there is no matching bound token at any level of prefixing (in which case it renders the key as is).

Note that in this structure, all of the information is in terminal nodes, and documents are seen as having no value, with each string acting as a link to a document. If this is a laborious representation, we can allow the nodes to hold value, in which case this system looks different.

### **A High Level Overview**

Our implementation consists of two core operations on three data structures to be further described below.

Nouns: **Document**, **Link**, **Token** (assumes the existence of List and Map)

Verbs: **Render_document**, **Render_key**, **Search_for**

A few things to consider:

I am approaching the object model as if it can have cycles. Also note that the label of any given key is path dependent. For example, in the graph:

--------a--------  
-------/--\\-------  
------b---c------  
-------\\---/------  
--------d--------

the keys in 'd' can have two names - those that contain prefixes from 'b', and those that contain prefixes from 'c'. In this current implementation, I am ignoring this path dependence and marking a node visited irregardless of path.

#### **Nouns: Data Structures**

Our first implementation will operate only on the level of these entities:

* A **Document**, which has only one type. It contains a value (a list of Tokens (described below))m and a link_map of type Map<String, [Token]>, or differently conceptualized, Map<String, Document([Tokens], {})>. Constructer: document(value :: [Token], link_map :: Map<String, [Token]>)
* A **Token**, which can be of type:
    1. **Literal**, which is a reference to an immediate value (int, string, bool, etc). Constructer: token(value :: Object)
    2. **Hanging**, is a link to an unknown value. This is what was previously referred to as a 'variable'. Constructer: token(variable :: String)
    3. **Linked**, which is a reference to a another Document. This should only appear in the Map. Constructer: token(reference :: String? Not sure about what a reference is)

I would also like to consider the idea of a Universe, which is defined by all possible combination of prefixed to totally deprefixed (empty string) values. **Note that the algorithm as currently described performs a greedy search, choosing to inhabit the first valid Universe it encounters on each execution of render_key.** Any formal verification / queries we answer can only refer to the items within the universe (although it can refer to how the universe can behave even with arbitrary input).

#### **Verbs: Functions**

* **Render_document**

  A Document is rendered by repeated dereferencing its value (a list of tokens). Each call of render_document involves a series of calls to render_key, described below (each call at a lower level of prefixing).

* **Render_key**

  Hanging Tokens can be considred as references to currently unknown tokens to be discovered at render time by the algorithm sketched below:

  1. Initial Execution: A depth first search, full prefix
  2. Recur: Deprefix. If base prefix has been reached (empty string), procede to step 3. If it has not been reached, return to step 1 (can be seen as recurring into less specific worlds).
  3. Base Case: No match found in any of the version of the Model's Universe (In this case, a Model's universe consists of all of its links named by each level of prefixing of the current key, holding all other keys constant). In this eventuality, a pointer to a terminal node with they key name as its value.

### **A More in Depth Definition (with Pseudocode)**

#### A (painfully insufficient) Primer on Functional Programming for the Uninitiated

Here I will provide a spec that looks slightly more like code. Also, I am nowhere near an expert in this, so please follow your curiosity on Google. There are plenty of great tutorials out there (I link to one in Haskell, a functional language, below the pseudocode).

For someone who is not familiar with functional programming, read the next paragraph. For those who are, feel free to skip to the next header (Data Structures). The pseudocode syntax isn't exactly Haskell, but its close enough that you should be able to make sense of it (its taken from pyret, the **functional language** taught in CS19 at Brown).

A quick introduction to the syntax used below: I wrote the pseudocode in a functional manner using recursion. I assumed no mutable values (any mutation is assumed to be a function that takes in the original structure and outputs the new one). The main syntax that may be confusing is the cases block (similar to guards / pattern matching in Haskell). The cases block can be seen as a special type of if statement, except it often depends solely on the structure of the data. The program matches the first 'pattern' it can find, and then executes that block.

For example, imagine counting items in a list. The list can be either empty (a pattern, 'mt') or still have values (also a pattern, here referred to as cons(head, rest)). If it still has values left, return 1 + the same function called on the rest of the list. If not, then you return the current counter. In the imperative approach, the list structure does not help you solve the problem, so we must define the solution in the method, whereas in the functional approach, the solution falls nicely out of the structure of the data.

Here's the example described above:

```python
#Imperatively
def list_length(list):
  '''
  Calculates the length of a list.
  '''
  length = 0
  for i in list:
    length += 1


#Functionally

#the data structure
data List:
  | cons(head :: Object, rest :: List)
  | mt

#the function
def list_length(list):
  '''
  Calculates the length of a list.
  '''
  cases(List):
    | populated =>
      return 1 + list_length(rest(list))
    | mt => return 0
```

Another example data structures would be:

```python
data Tree:
  | populated(value :: Object, left_child :: Tree, right_child :: Tree)
  | leaf(value :: Object)
```

As a [Haskell tutorial](http://learnyouahaskell.com/chapters) I am exploring succinctly said (emphasis mine), "Recursion is important to [functional programming] because unlike [in imperative programming], you do computations in [functional programming] by declaring **what something is** instead of declaring **how you get it**." If you are interested in diving more deeply into functional concepts, I would highly recommend following the link above. [Here](http://learnyouahaskell.com/chapters) it is again.

Recursion is a central concept in functional programming. Each recursive function can be seen as performing the same computation on subsets of the structure and then combining all the values to calculate the final answer. The structure of the data is therefore very important and can also end up driving the algorithm. I haven't programmed functionally in a while (since freshman year), but I remember that Geoffrey mentioned Haskell as an option, and I believe that the core algorithm would be well suited to a functional approach. I've been reading through some Haskell tutorials and believe I have enough of a grasp of the language to code an MVP of the core algorithm in the next step of this process. Thoughts would be appreciated on this point, because starting in Haskell may force the others to learn it as well, which could be burdensome. Either way, beginning with a functional approach will really help me understand the problem, so even if we switch to python, we can use the structure defined here.

#### **Data Structures (with Pseudocode)**

```python
data Document:
  '''
  Represents a Node on the Prose Object Graph. Note that the "value" field is
  equivalent to Model.root in the previous implementation.

  Types
  -----
  terminal: A node that has no outgoing connections. This node contains text,
    and is the only node in the Prose Object Graph that has a non relational
    value. Each value in link_map can be seen as its own document.
  
  linked: A node that has outgoing links. Subfields:
  '''
  | linked(value :: [Token], link_map :: {key1 : [Token], ...})

data Token:
  '''
  Represents a Token (a value) on the Prose Object Graph. The closest analogue
  in the previous implementation was a variable. I expanded its definition to include
  normal strings (it makes the recursion easier).

  Types
  -----
  literal: A token that has no references within it.
  
  linked: A token that is a reference to another document.

  mt: A token with no value. Useful for deprefixing.
  '''
  # I need to consider how to implement this with extensibility to scripts in mind
  # This needs to be able to handle code as well as text...
  # a literal value (int, bool, string, etc)
  | literal(value :: Object)
  # a reference to a document
  | linked(reference :: String)
  # a reference to a token somewhere else
  | hanging(variable :: String)
  # an empty token. Has no value
  | mt
```

#### **Functions (with Pseudocode)**

```python
def render_document(document, prefixes):
  '''
  Render a document. Note that document.value is my version of root.map.
  Also note that prefixes is stored in recent to least recent, meaning you need
  to reverse when composing it into a single string.

  Parameters
  ----------
  document: the document to render

  Returns
  -------
  The string representing the rendered document.
  '''
  tokens = document.value
  #dereference the next variable
  cases(List) tokens:
    | [] => return '' #return if tokens is an empty list. This is the base case
    | (head, rest) =>
      rest_of_document = document(rest, token_map, name)
      cases(Token) head:
        | literal =>
          #if the token is literal, then we return its value + the rest of the rendered map
          return head.value + render_document(rest_of_document, prefixes)
        | hanging =>
          #add the necessary prefixes to the variable
          head = fold(reverse(prefixes), lambda x, y: x + y) + head
          prefixes, dereferenced, call_number, matched_document =
            dereference_token(document, next_variable, prefixes, 0)
          # the dereferenced variable may itself contain a reference which we must render
          # we create a modified version where its value to render is the dereferenced value.
          modified_document = document(dereferenced, document.link_map)
          return render_document(modified_document, prefixes) +
            render_document(rest_of_document, prefixes)
        | linked =>
          render_document(head.reference, )

def dereference_token(document, token, prefixes, call_number):
  '''
  Dereferences a token.

  Parameters
  ----------
  document: the document we are dereferencing in
  token: the token to be dereferenced
  prefixes: the prefixes to be added during the dereferencing
  call_number: the depth of the current call in the recursive call stack

  Returns
  -------
  (prefixes, best_dereferenced_value, call_number): The prefixes that were used
    to match, the value that was matched, and the call_number.
  '''
  # return the best matched key and the prefixes that it was matched with
  (prefixes, best_choice) = scan_options_in_current_doc(link_map, token, prefixes)
  if len(prefixes) == depth:
    # if full prefixed answer no need to recur as everything in the subtree
    # is of lower priority in the ordering
    return (prefixes, best_choice, call_number, document)
  else:
    best_list = [(mt, mt, call_number)]
    # get all keys that are a reference to another document
    # NOTE: This does not capture references to documents that are within
    # larger lists of token. For example, it misses key="Here is me!" + [/path/to/myself]
    neighbors == [k for k in link_map.keys where k.is_linked]
    # recur on each neighbor
    for neighbor in neighbors:
      best_list = cons(dereference_token(neighbor), best_list)
    # fold to return the option with the best prefix length
    return fold(best_list, comparison_function(x, y))

def scan_options_in_current_doc(link_map, token, prefixes):
  '''
  Finds the best match in the current document. Recursively deprefixes until
  a match is found.

  Parameters
  ----------
  link_map: The key/value map of the current document being searched
  token: The token to be dereferenced
  prefixes: The prefixes up to be used while matching.

  Returns
  -------
  (prefixes, dereferenced_value): The prefixes that were used in the matching and
    the dereferenced value.
  '''
  squashed_prefixes = fold(reverse(prefixes), lambda x, y: x + y)
  # prefixed token to match
  token = squashed_prefixes + token
  pref_keys = map(link_map.keys, lambda x: squashed_prefixes + x)
  # dictionary matching prefixed key to original key
  k_to_pk = {p_k : p for p_k, k in link_map.keys, pref_keys}
  # we matched!
  if token is in pref_keys:
        key = k_to_pk[token]
        return (prefixes, link_map[key])
  else:
    # there was no match
    # recur if there are still prefixes left. Else return mt.
    cases(List) prefixes:
      | mt =>
        return(mt, mt)
      | (head, rest) =>
        return scan_options_in_current_doc(link_map, token, rest)

def comparison_function(x, y):
  '''
  Compares x and y. Returns longest prefix. If tiebreaker, returns the earlier match.
  Remember that x and y are tuples (prefixes, best_choice, call_number). Also note
  that this works even if one or both of x and y are of the form (mt, mt, call_number).

  Parameters
  ----------
  x, y: tuples of the form (prefixes, chosen_dereference_value, call_number) to be compared.

  Returns
  -------
  The value with higher priority (less deprefixed, or if even prefix length, earlier called)
  '''
  x_prefix, x_choice, x_call_number = x[0], x[1], x[2]
  y_prefix, y_choice, y_call_number = y[0], y[1], y[2]
  # If x is less deprefixed then y, return x
  if len(x_prefix) > len(y_prefix):
    return x
  # If there is a tie on prefix length, return the earlier seen value
  elif len(x_prefix) == len(y_prefix):
    if x_call_number < y_call_number:
      return x
    else:
      return y
  # Else, either y's prefix is longer than x's or they are tied on prefix length
  # but y was seen earlier
  else:
    return y
```

### Moving Forward: a Roadmap

Firstly, we should establish a fresh repo. We can definitely add previous work (Perl code, Geoffrey's work) as seperate branches, but I think it would be nice to have a clean canvas (in terms of Git history) to begin with (on second thought, I'm fairly indifferent to this point. Up to you guys). We should also discuss the particulars of how we develop in the repo and merge code to master - Test Driven Development, a Checkstyle, and the expected branching / commit message / pull request behavior. Then, we should define a more detailed long term roadmap. The one I define below has an incredibly large scope, but each step is a subset of the last and a totally functional project in and of itself. However, we should aim to be developing with future steps in mind, even if we don't believe we will complete them all. Currently I see it as:

1. Choose a language for the first attempt at coding the algorithm (**I'm leaning towards Haskell**)
2. Finish the algorithm (including tests). Note that this step does not include any complex I/O like reading and writing to files - it will just be predefined structures held in memory, *maybe* written to a text file for small proof of concept.
3. Provide an implementation of the algorithm. I think the filesystem based approach from the original attempt would be a good place to start.
4. Design an interface. Write a simple front end (**I'm leaning towards React**), wrap the code from step 3 in a server. Test on localhost.
5. Host the code and interface. Possibly consider containerizing it and deploying it as a service using Docker, Kubernetes, and AWS/Google Cloud/IPFS.
6. Extend functionality to include version control. This means Git!
7. Second round of UX - a great goal (obviously a stretch) is to emulate the experience of google docs. This means Git + concurrent editing of the same working directory!
8. Add a Query language on top of the Prose Object Model. Just like SQL opened up a whole new universe of questions (arbitrary queries on relational data) we can open up a whole universe of questions (how does this mathematical model of the graph *behave when something happens*). If Acme goes bankrupt, what happens to my bond holdings? If England wins the game, what happens to my account balance? Etc. This should look and feel just like SQL. We can also consider what mathematical proofs we can provide about the behavior of the graph aka **formal verification** (very large stretch, would take a lot of learning). We should also consider this when implementing the algorithm. Any desire to **reason formally about the Prose Object Model** may require a **functional approach**.
9. Add integration functionality to arbitrary ledgers. Any ledger that fulfills this API should be able to implement and interact with the Prose Object Model. Examples of ledgers: A SQL database, VISA, Ethereum, Bitcoin. We are just an intermediate layer - we will **push trust to the edges**.
10. Add specific implementations of these ledger APIs. I'm thinking we first do a SQL database implementation and an Ethereum implementation.