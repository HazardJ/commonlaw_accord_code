# Preliminary Spec

I consider this document a first attempt at defining the structure of the coding portion
of the GISP.

## A Theoretical Aside

I apologize for the pseudophilosophical ramble that follows - stuff like this helps me conceptualize the abstract system that I am going to implement.

The prose object model is a way of representing a legal document as a node on a graph (in the mathematical sense of the word). The document is defined by other nodes it references on a graph (this abstraction is described in Gabriel’s Monads, and is the abstraction that inspired PageRank as well). The cool thing about the prefix system (and the instantiation construct) is that *they provide a stronger differentiation of a node’s worldview*. Not only is a node’s view affected by its location in the graph (aka my perspective is shaped by location on earth. If I’m in Paris, I see the Eiffel Tower, if I’m in NYC, I see the Empire State), but also that prefixing provides a *different etymology*, and instantiation provides a *different ontology*. Not only does the node have a unique view into the world – it has its *own unique world* that only it inhabits. A node’s world is defined by its **location in the graph** (its edges) and **its prefixes**. Let me continue the geographic analogy, except in this case I will add the concept of prefixing back in. Consider again two nodes – one Parisian, one New Yorker. Not only are their perspectives different spatially (one sees Paris, one sees New York), but they literally live in different worlds. The Parisian operates with a certain set of prefixes (French, Parisian, etc) that defines how it sees things. Consider the object my_city. Both reference the file “city”, but one sees Parisian.city and the other sees New_Yorker.city. Same object, different instantiation. They both reference the my_city node, but instantiate it differently. Same graph, same platonic objects, but different names, different instantiations. Each instantiation can be seen as creating a new node
that is based of a Platonic Ideal (the uninstantiated my_city object) but is itself a unique entity (a **Document**, or equally, a **Model**) in the graph.

With that philosophical aside over with, I will now describe how I believe we should architect the program. I haven’t looked deeply at the Perl implementation (I’ve never used the language and don’t really have time to learn right now), but I’m guessing that it’s too tightly tied to that specific implementation of the Prose Object Model (specifically file based architecture). In my opinion, this makes the interface confusing – the user (even if it is only interfaced with through its method contracts)should be allowed to first think in Prose Object Graph, and only then consider specific implementations.

## The Spec

### Defining the Abstract Structure

Note: I am still not sure if I want to think about this functionally are object orientally yet, so forgive me for mixing paradigms / providing some ambiguity.

I propose that we begin by defining a set of function contracts that defines a Prose Object Model in the abstract. Each node shall be, in this document, hereafter referred to as a **Document** (in the spirit of the contracts as artificats doc you guys linked). Each Model contains a value (its text content which may be sprinkled with keys), and an edge list, and an edge lookup (where keys are linked to edges). Note that the value is not actually seperate, but are an ordered list of *edges to terminal nodes of the graph*. Terminal nodes are nodes that have no outgoing edges, and are the only nodes in the graph with value (a string, a number, etc). Non terminal nodes will only contain a list of keys and a list of edges. Our first implementation should consider keys in the abstract (as a *symbolic link to some Document*, not a string enclosed in braces), and values in the same way (not as a string or filepath etc, but a *pointer to another Document*). In this case, the node that contains “my age is {jake.name}” will actually contain (in its edge list, in addition to other references) a reference to a node that contains “my age is “ and a symbolic link that contains {jake.name} (if this reference to this link cannot be found, it can be considered a reference to a string ‘{jake.name}’). Note that I have not yet thought out how we will represent the difference between a link determined by a key (which involves prefixing) vs a link w/ no key. It is possible that a keyed link can just be considered a subclass of Edge. 

### Functions: A High Level Overview

Our implementation consists of two fundamental operations on two data structures to be further described below.

Verbs: **Render**, **Deprefix**

Nouns: **Document** (Linked_Document, Terminal_Document) and **Link**

#### Verbs

##### Render

A Document is rendered by repeated deprefixing. Deprefix is recursively called on a Linked_Document until it becomes a Terminal_Document (Base Case).

##### Deprefix

Links can be considred as **links** to currently unknown edges to be discovered at render time by the algorithm sketched below:

1. Initial Execution: A depth first search, full prefix
2. Recur: Deprefix. If base prefix has been reached (empty string), procede to step 3. If it has not been reached, return to step 1 (can be seen as recurring into less specific worlds).
3. Base Case: No match found in any of the version of the Model's Universe (In this case, a Model's universe consists of all of its links named by each level of prefixing of the current key, holding all other keys constant). In this eventuality, a pointer to a terminal node with they key name as its value.

Our first implementation will operate only on the level of these entities:

1. A **Document**, which is subclassed by a **Linked Document** and a **Terminal Document**
2. A **Link**, which is a reference to another **Document**

I would also like to consider the idea of a Universe, which is defined by all possible combination of prefixed to totally deprefixed (empty string) values. **Note that the algorithm as currently described performs a greedy search, choosing to inhabit the first valid Universe it encounters on each execution of deprefix.**

### A More Technical Definition

Here I will provide a spec that looks slightly more like code.

#### Main Verbs: Function Contracts (in Statically Typed syntax)

For any of you who did CS19 this came out looking like Pyret lol.

def render_document(document : Document):
  cases:
    document.is_Linked -> deprefix(document)
    document.is_Terminal -> return document.value

def deprefix(document : Document):
  to_render = get_next_key(document)
  cases:
    traverse(next_linked(to_render)).is_Terminal -> return render_terminal(document)
    traverse(next_linked(to_render)).is_Linked -> render(document)

def render_terminal(terminal : Terminal):
  assert document.is_Terminal

#### Main Nouns: Defining Data Structures

NOTE: I am not sure if I am happy with the document data structure. Terminals are still documents, and it therefore feels wrong to break it out into a different structure (it is a special document with links to only strings). But then I have an issue - how do I represent a value in my current Document structure?

Document {
  links = [list_of_links]
  keys = {key1 : link_in_links, ...}
  name = my_name
}

Terminal {
  value = a_string
}

Link {
  start: starting_Document
  destination: dest_Document
}

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