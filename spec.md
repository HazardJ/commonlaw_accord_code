# **Preliminary Spec**

I consider this document a first attempt at defining the structure of the coding portion
of the GISP.

## **A Theoretical Aside**

I apologize for the pseudophilosophical ramble that follows - stuff like this helps me conceptualize the abstract system that I am going to implement.

The prose object model is a way of representing a legal document as a node on a graph (in the mathematical sense of the word). The document is defined by other nodes it references on a graph (this abstraction is described in Gabriel’s Monads, and is the abstraction that inspired PageRank as well). The cool thing about the prefix system (and the instantiation construct) is that *they provide a stronger differentiation of a node’s worldview*. Not only is a node’s view affected by its location in the graph (aka my perspective is shaped by location on earth. If I’m in Paris, I see the Eiffel Tower, if I’m in NYC, I see the Empire State), but also that prefixing provides a *different etymology*, and instantiation provides a *different ontology*. Not only does the node have a unique view into the world – it has its *own unique world* that only it inhabits. A node’s world is defined by its **location in the graph** (its edges) and **its prefixes**. Let me continue the geographic analogy, except in this case I will add the concept of prefixing back in. Consider again two nodes – one Parisian, one New Yorker. Not only are their perspectives different spatially (one sees Paris, one sees New York), but they literally live in different worlds. The Parisian operates with a certain set of prefixes (French, Parisian, etc) that defines how it sees things. Consider the object my_city. Both reference the file “city”, but one sees Parisian.city and the other sees New_Yorker.city. Same object, different instantiation. They both reference the my_city node, but instantiate it differently. Same graph, same platonic objects, but different names, different instantiations. Each instantiation can be seen as creating a new node
that is based of a Platonic Ideal (the uninstantiated my_city object) but is itself a unique entity (a **Document**, or equally, a **Model**) in the graph.

With that philosophical aside over with, I will now describe how I believe we should architect the program. I haven’t looked deeply at the Perl implementation (I’ve never used the language and don’t really have time to learn right now), but I’m guessing that it’s too tightly tied to that specific implementation of the Prose Object Model (specifically file based architecture). In my opinion, this makes the interface confusing – the user (even if it is only interfaced with through its method contracts)should be allowed to first think in Prose Object Graph, and only then consider specific implementations.

## **The Spec**

### **Defining the Abstract Structure**

Note: I am still not sure if I want to think about this functionally are object orientally yet, so forgive me for mixing paradigms / providing some ambiguity.

I propose that we begin by defining a set of function contracts that defines a Prose Object Model in the abstract. Each node shall be, in this document, hereafter referred to as a **Document** (in the spirit of the contracts as artificats doc you guys linked). Each Model contains a value (its text content which may be sprinkled with keys), and an edge list, and an edge lookup (where keys are linked to edges). Note that the value is not actually seperate, but are an ordered list of *edges to terminal nodes of the graph*. Terminal nodes are nodes that have no outgoing edges, and are the only nodes in the graph with value (a string, a number, etc). Non terminal nodes will only contain a list of keys and a list of edges. Our first implementation should consider keys in the abstract (as a *symbolic link to some Document*, not a string enclosed in braces), and values in the same way (not as a string or filepath etc, but a *pointer to another Document*). In this case, the node that contains “my age is {jake.name}” will actually contain (in its edge list, in addition to other references) a reference to a node that contains “my age is “ and a symbolic link that contains {jake.name} (if this reference to this link cannot be found, it can be considered a reference to a string ‘{jake.name}’). Note that I have not yet thought out how we will represent the difference between a link determined by a key (which involves prefixing) vs a link w/ no key. It is possible that a keyed link can just be considered a subclass of Edge.

A non-terminal document only holds and an identifier, tokens (terminal and hanging), and links. It holds the tokens in one basket, and the links in another - a value basket and a map basket respectively. The value basket is the body of the document, which is to be rendered in sequential order. When rendering a terminal token, you can load the value of the terminal document directly. When loading a hanging token, a search must be exectuted. This is where the map comes into play.

The map basket holds, in its values, all of the other documents that document knows about. These are represented as links. The keys of the map are bound tokens. The values of the map may be conceptualized as roads, whereas the keys of the map are road signs.

When a hanging token is referenced, a search commences. It ends when it finds a matching bound token or exhausts the available subgraph and knows that there is no matching bound token at any level of prefixing (in which case it renders the key as is).

Note that in this structure, all of the information is in terminal nodes, and documents are seen as having no value, with each string acting as a link to a document. If this is a laborious representation, we can allow the nodes to hold value, in which case this system looks different.

### **Functions: A High Level Overview**

Our implementation consists of two fundamental operations on two data structures to be further described below.

Verbs: **Render_document**, **Render_key**, **Search_for**

Nouns: **Document**, **Link**, **Map**

#### **Verbs**

##### **Render_document**

A Document is rendered by repeated deprefixing. Deprefix is recursively called on a Linked_Document until it becomes a Terminal_Document (Base Case).

##### **Render_key**

Links can be considred as **links** to currently unknown edges to be discovered at render time by the algorithm sketched below:

1. Initial Execution: A depth first search, full prefix
2. Recur: Deprefix. If base prefix has been reached (empty string), procede to step 3. If it has not been reached, return to step 1 (can be seen as recurring into less specific worlds).
3. Base Case: No match found in any of the version of the Model's Universe (In this case, a Model's universe consists of all of its links named by each level of prefixing of the current key, holding all other keys constant). In this eventuality, a pointer to a terminal node with they key name as its value.

Our first implementation will operate only on the level of these entities:

1. A **Document**. It can be of type:
    1. **Terminal Document**, which is a document containing only a String (and eventually Turing complete code).
    2. **Linked Document**, which is a document containing a list of tokens as well as a Map of {key : link, ...}. The list of links
2. A **Link**, which is a reference to another **Document**. It can be of type:
    1. **Terminal Link**, which is a reference to a Terminal Document
    2. **Hanging Link**, is a link to an unknown value. This is what was previously referred to as a 'key'.
    3. **Linked Link**, which is a reference to a nother Linked Document. This should only appear in the Map.
3. A **Map**

I would also like to consider the idea of a Universe, which is defined by all possible combination of prefixed to totally deprefixed (empty string) values. **Note that the algorithm as currently described performs a greedy search, choosing to inhabit the first valid Universe it encounters on each execution of render_key.** Also note that the current approach (depth first search) works will in a directory structure (which is tree-like) but may run into serious issues in a more complicated graph, especially one that is fully traversable and has cycles.

### **A More Technical Definition**

Here I will provide a spec that looks slightly more like code.

A quick introduction to the syntax used below (stolen from pyret, the **functional language** taught in CS19 at Brown). The language has no mutable values. The central construct of the language is the 'cases' block. The cases block can be seen as a special type of if statement, except it is one that depends solely on the structure of the data. For example, imagine counting items in a list. The list can be either empty or still have values left. If it still has values left, return 1 + the same function called on the rest of the list. If not, then you return the current counter. For example:

```python
#the data structure
data List:
  | populated(head :: Object, rest :: List)
  | mt(None)

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

Other example data structures would be:

```python
data Tree:
  | populated(left_child :: Tree, right_child :: Tree)
  | leaf

data Graph:
  | Node(value :: Object, neighbors :: List)

```

Recursion is a central concept in functional programming. Each recursive function can be seen as performing the same computation on subsets of the structure and then combining all the values to calculate the final answer. The structure of the data is therefore very important and can also end up driving the algorithm. I haven't programmed functionally in a while (since freshman year), but I remember that Geoffrey mentioned Haskell as an option, and I believe that the core algorithm would be well suited to a functional approach. Even if we end up using python, a functional brainstorm can serve as great inspiration.

#### **Main Verbs: Function Contracts (in Statically Typed syntax)**

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
  The string representing the fully rendered document
  '''
  cases(Document) document:
    | terminal => return ''
    | linked =>
      rest_of_document = Document(document.token_list, rest(document.link_map),
        document.name)
      next_token = head(document.token_list)
      render_next_token(next_token, document.link_map, 0) + render_document(rest_of_document)

def render_next_token(next_token :: Token, link_map :: Map, deprefix_count):
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
  cases(Link) next_token:
        | literal => return token.value
        | hanging =>
          # Do a horizontal search of the current file
          if token.value is in link_map.keys:
            return link_map[token.value]
          else:
            # Do a DFS, starting with full prefixing
            dereferenced = search_for(next_token, link_map, 0, [], link_map)
            # NOTE: HOW DO WE KNOW WHEN WE HAVE REACHED MAX DEPREFIX
            # TODO: Fix this hacky solution
            if dereferenced == -1:
              return next_token
            elif derefrences == -2:
              return render_next_token(next_token, link_map, deprefix_count + 1)
            else:
              return dereferenced

def search_for(token :: Token.hanging, link_map :: Map, prefixes :: List<String>,
  deprefix_count :: Int, original_link_map :: Map):
  '''
  Searches for the next terminal node that matches the pattern referenced in
  key.
  NOTE: I do not think this handles cycles yet... That may be tricky. I believe
  we will have to pass a path parameter AND give each node a unique identifier
  (a hash).
  NOTE: This is currently super inefficient (it doesn't cut out impossible branches
  while prefixed, nor does it collect all possible answers in a single execution and
  stop when it knows its found the best one.)
  '''
  cases(link_map):
    | populated =>
      next_link = head(link_map.values)
      neighbor = next_link.destination
      keys = neighbor.link_map.keys
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
        #go to next depth and repeat
        result = search_for(token, neighbor.link_map.destination.link_map, deprefix_count, original_link_map)
        if result is not None:
          return result
        else:
          #if we didn't find anything, search horizontally
          search_for(token, rest(link_map), prefixes, deprefix_count, original_link_map)
    #will this happen in the right order... No it will not, has to be called from top
    | mt => return None #NOTE: Need some way to move knowledge about prefixing limit up the stack


def render_terminal(terminal :: Terminal):
  assert document.is_Terminal
```

#### **Main Nouns: Defining Data Structures**

```python
data Document:
  '''
  Represents a Node on the Prose Object Graph

  Types
  -----
  terminal: A node that has no outgoing connections. This node contains text,
    and is the only node in the Prose Object Graph that has a non relational
    value.

    Subfields
    ---------
    value: the value of the node. Currently only a string. In the future, should
      be a Turing Complete script.
  
  linked: A node that has outgoing links. Subfields:
    value: a
  '''
  | terminal(value :: String) # We may expand this to include other things,
    # such as code
  | linked(value :: [list_of_links], link_map :: {key1 : link_in_links, ...},
      name :: string)

data Link:
  '''
  Represents an Edge on the Prose Object Graph
  '''
  | linked(start :: Document.linked, destination :: Document.linked)
  | terminal(start :: Document.linked, destination :: Document.terminal)

data Token:
  '''
  Represents a Token (a value) on the Prose Object Graph
  '''
  # I need to consider how to implement this with extensibility to scripts in mind
  # This needs to be able to handle code as well as text...
  | literal(value :: Object)
  | hanging(reference :: String, prefixes :: List)
  | mt
```

### Moving Forward: a Roadmap

Firstly, we should establish a fresh repo. We can definitely add previous work (Perl code, Geoffrey's work) as seperate branches, but I think it would be nice to have a clean canvas (in terms of Git history) to begin with. We should also discuss the particulars of how we develop in the repo and merge code to master - Test Driven Development, a Checkstyle, and the expected branching / commit message / pull request behavior. Then, we should define a more detailed long term roadmap. The one I define below has an incredibly large scope, but each step is a subset of the last, and a totally functional project in and of itself. We should aim to be developing with extensionality in mind. Currently I see it as:

1. Choose a language for the first conceptual attempt
2. Finish the conceptual kernel of the project (including tests). Note that this step does not include any complex I/O like reading and writing to files - it will just be predefined structures held in memory, *maybe* written to a text file for small proof of concept.
3. Provide an implementation of the kernel. I think the filesystem based approach from the original attempt would be a good place to start.
4. Provide an interface. Write a simple front end in React. Test on localhost.
5. Host the code and interface. Possibly consider containerizing it and deploying it as a service using Docker, Kubernetes, and AWS/Google Cloud/IPFS.
6. Extend functionality to include version control. This means Git!
7. Second round of UX - a great goal (obviously a stretch) is to emulate the experience of google docs. This means Git + concurrent editing of the same working directory!
8. Add a Query language on top of the Prose Object Model. Just like SQL opened up a whole new universe of questions (arbitrary queries on relational data) we can open up a whole universe of questions (how does this mathematical model of the graph *behave when something happens*). If Acme goes bankrupt, what happens to my bond holdings? If England wins the game, what happens to my account balance? Etc. This should look and feel just like SQL.
9. Add integration functionality to arbitrary ledgers. Any ledger that fulfills this API should be able to interact with the Prose Object Model. Examples of ledgers: A SQL database, VISA, Ethereum, Bitcoin. We are just an intermediate layer - we will **push trust to the edges**.
10. Add specific implementations of these ledger APIs. I'm thinking we first do a SQL database implementation and an Ethereum implementation.