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

the keys in 'd' can have two names - those that contain prefixes from 'b', and those that contain prefixes from 'c'.

#### **Nouns: Data Structures**

Our first implementation will operate only on the level of these entities:

* A **Document**. It can be of type:
    1. **Terminal Document**, which is a document containing only a String (or eventually Turing complete code, although I'm not sure how we'd represent this yet).
    2. **Linked Document**, which is a document containing a list of tokens as well as a Map of {key : link, ...}. The list of links
* A **Link**, which is a reference to another **Document**. It can be of type:
    1. **Terminal Link**, which is a reference to a Terminal Document
    2. **Hanging Link**, is a link to an unknown value. This is what was previously referred to as a 'key'.
    3. **Linked Link**, which is a reference to a nother Linked Document. This should only appear in the Map.
* A **Token**

I would also like to consider the idea of a Universe, which is defined by all possible combination of prefixed to totally deprefixed (empty string) values. **Note that the algorithm as currently described performs a greedy search, choosing to inhabit the first valid Universe it encounters on each execution of render_key.** Any formal verification / queries we answer can only refer to the items within the universe (although it can refer to how the universe can behave even with arbitrary input).

#### **Verbs: Functions**

* **Render_document**

  A Document is rendered by repeated dereferencing hanging tokens. Render_document is recursively called on a Linked_Document until it becomes a Terminal_Document (Base Case). Each call of render_document involves a series of calls to render_key, described below (each call at a lower level of prefixing).

* **Render_key**

  Hanging Tokens can be considred as references to currently unknown tokens to be discovered at render time by the algorithm sketched below:

  1. Initial Execution: A depth first search, full prefix
  2. Recur: Deprefix. If base prefix has been reached (empty string), procede to step 3. If it has not been reached, return to step 1 (can be seen as recurring into less specific worlds).
  3. Base Case: No match found in any of the version of the Model's Universe (In this case, a Model's universe consists of all of its links named by each level of prefixing of the current key, holding all other keys constant). In this eventuality, a pointer to a terminal node with they key name as its value.

### **A More in Depth Definition (with Pseudocode)**

#### A (painfully insufficient) Primer on Functional Programming for the Uninitiated

Here I will provide a spec that looks slightly more like code. Also, I am nowhere near an expert in this, so please follow your curiosity on Google. There are plenty of great tutorials out there (I link to one in Haskell, a functional language, below the pseudocode).

For someone who is not familiar with functional programming, read the next paragraph. For those who are, feel free to skip to the next header (Data Structures). The pseudocode syntax isn't exactly Haskell, but its close enough that you should be able to make sense of it (its taken from pyret, the **functional language** taught in CS19 at Brown).

A quick introduction to the syntax used below: I wrote the pseudocode in a functional manner using recursion. I assumed no mutable values (any mutation is assumed to be a function that takes in the original structure and outputs the new one). The central construct of the language is the 'cases' block (similar to guards / pattern matching in Haskell). The cases block can be seen as a special type of if statement, except it often depends solely on the structure of the data. The program matches the first 'pattern' it can find, and then executes that block.

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
    value.
  
  linked: A node that has outgoing links. Subfields:
  '''
  | terminal(value :: Token.literal) # We may expand this to include other things,
  | linked(value :: [Token], token_map :: {key1 : token1, ...},
      name :: string)

data Token:
  '''
  Represents a Token (a value) on the Prose Object Graph

  Types
  -----
  literal: A token that has no references within it.
  
  linked: A token that has references.

  mt: A token with no value. Useful for deprefixing.
  '''
  # I need to consider how to implement this with extensibility to scripts in mind
  # This needs to be able to handle code as well as text...
  | literal(value :: Object)
  | linked(reference :: String, prefixes :: List)
  | mt
```

#### **Functions (with Pseudocode)**

```python

def render_document(document :: Document):
  '''
  Render key by key until the document is terminal, at which time we return its
  value.

  Parameters
  ----------
  document: a Document to render

  Returns
  -------
  The string representing the fully rendered document.
  NOTE: This can be modified to return a document with no links holding only a fully
    rendered value.
  '''
  rest_of_document = Document(document.token_list, rest(document.link_map),
    document.name)
  next_token = head(document.token_list)
  render_next_token(next_token, document.link_map) + render_document(rest_of_document)

def render_next_token(next_token :: Token, link_map :: Map, prefixes :: List<String>):
  '''
  Render the next key in the document.

  Parameters
  ----------
  document: The document being actively traversed
  original_document: The original document, kept to return in case no match for
    the key is found.
  
  Returns
  -------
  The document with the given key rendered.
  '''
  cases(Token) next_token:
        | literal => return token.value
        | hanging =>
          search_for(next_token, link_map, prefixes)

def search_for(token :: Token.hanging, link_map :: Map, prefixes :: List<String>,
  visited :: List<Nodes>):
  '''
  Searches for the next terminal node that matches the pattern referenced in
  key. NOTE: This is currently super inefficient (it doesn't cut out impossible branches
  while prefixed, nor does it collect all possible answers in a single execution and
  stop when it knows its found the best one.) Both the proposed accumulating method and
  the looping method have the same worst case performance, so I would assume that that
  looping method is better b/c it has better average case as well.

  Parameters
  ----------
  token: The hanging token we are attempting to match to a key.
  link_map: The map of links (both terminal and linked)
  prefixes: holds all the prefixes collected up until this point.
  deprefix_count: states how many of the prefixes to discard on each attempt to
    match
  visited: A list that stores all of the visited nodes. Note that this requires that
    each node is unique identifiable (we can just use the uniqueness of pointer values,
    or we can hash the link_map with the terminal_list).
  original_link_map: Holds the original link map passed. Used for recurring.

  Returns
  -------
  The dereferenced token (if found), or the original hanging token otherwise.
  '''
  cases(link_map):
    | populated =>
      next_link = head(link_map.values)
      neighbor = next_link.destination
      # check if we've already visited this node with the same set of prefixes
      have_seen = is_in(neighbor, visited, prefixes)
      if have_seen:
        return search_for(token, rest(link_map), prefixes, deprefix_count, visited)
      else:
        #add neighbor / prefix pair to visited
        cons((neighbor, prefixes), visited)
      keys = neighbor.link_map.keys
      # PREFIXING + DEPREFIXING LOGIC
      # removes the most recent (rightmost) deprefix_count prefixes from prefixes
      prefixes = remove_last_n(prefixes, deprefix_count)
      # condense the prefix list into a single string
      prefix = fold(prefixes, lambda x, y: x + y)
      # add the prefix to each key
      prefixed_keys = map(keys, lambda x: prefix + x)
      #if we find a match
      if token.value is in prefixed_keys:
        return neighbor.link_map[token.value]
      else:
        # go to next depth and repeat
        result = search_for(token, neighbor.link_map.destination.link_map, deprefix_count, visited)
        if result is not None:
          # we found a match in the sub_tree!
          return result
        else:
          # if we didn't find anything, search horizontally
          return search_for(token, rest(link_map), prefixes, deprefix_count, visited)
    | mt => return None
      #NOTE: Need some better way to move knowledge about prefixing limit up the stack
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