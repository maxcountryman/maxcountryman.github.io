---
layout: post
title: "Data Structures in Clojure: Binary Search Trees"
date: 2014-04-09 19:26:57
---

## Trees Everywhere
So far we have talked about two fundamental and pervasive data structures:
[linked lists][0] and [hash tables][1]. Here again we discuss another important
data structure and one that you will find is quite common: trees. Trees offer
a powerful way of organizing data and approaching certain problems. In
particular, searching and traversal. Whether you know it or not, you no doubt
use trees in your programs today. For instance, Clojure's vectors are backed by
a special kind of tree!

Here we will construct our own tree, just like with our linked list and hash
table implementations. Specifically, our tree will be a kind of tree known as
a Binary Search Tree (BST). Often when someone says tree, they mean a BST.

We will look the basic structure of our tree, how we insert things into it, and
how we find them again. Then we will explore traversing, and finally, removing
nodes. At the end of this tutorial you will have a basic, functioning Binary
Search Tree, which will be the basis for further explorations later on in this series.

<p align="center">
    <img src="http://upload.wikimedia.org/wikipedia/commons/8/8d/The_Ash_Yggdrasil_by_Friedrich_Wilhelm_Heine.jpg" title="Yggdrasil: A very important tree">
</p>

### Taxonomy of a Tree
Trees are hierarchical structures composed of nodes. You probably are already
familiar with this concept if you have ever dealt with a filesystem. The
structure is a tree, where directories are parents of subdirectories.

A root node represents the beginning of the tree. Nodes below the root
are called children. Each child has a single parent and may have children of
its own, just like subdirectories in your filesystem.

Subtrees are contained below the root. A subtree is any partial representation
of the larger, total tree and is itself a tree. Edges from the root, connect it
to all subtrees.

The height of a tree is the number of levels of nodes it contains, excluding the
root node. So for instance, in the example below, the tree has a height of two.

The relationship between these nodes and their edges forms a visual structure
that looks a bit like an upside down organic tree, where the root is the trunk
and the branches are the edges to leaves. Of course this is not always true,
but by way of explanation, this is one reason we refer to them as trees.

<p align="center">
    <img src="/images/simple-tree.png" title="Simple Tree">
</p>

Actually, we have already seen a tree. If we think back to our
[linked list implementation][0]: Consider that the head of our list is a node
which contains a pointer to the next node. If we follow these links, we
eventually end up at the tail. The head is equivalent the root of the tree
and each node in the list is a node in the tree with only one child. The tail
of the linked list would be referred to as a leaf in tree parlance.

<p align="center">
    <img src="/images/ll-tree.png" title="A linked list and a tree: both trees!">
</p>

Now linked lists are a very lopsided example of a tree. Typically trees will
have two or more branches from a given node. The number of edges a tree has
will often relate to the performance guarantees they provide and the algorithms
we can use to manipulate them. For instance, [Clojure's vectors][2] are in fact
shallow trees each with 32 children.

### Binary Search Tree
Binary Search Trees are defined as having two branches or edges, we call this
the branching factor. These trees also must satisfy the following properties
(known as the Binary Search Tree Property):

1. The left subtree of a given node must only contain keys that are less
than the parent node's.

2. The right subtree of a given node must only contain keys that are more
than the parent node's.

3. The left and right subtrees must themselves be binary search trees, i.e.
they must have two edges that satisfy the above conditions.

4. Finally there cannot be duplicate nodes within the tree.

Another way to think about these rules is that a BST can be defined as a root
node with two children which are themselves defined in the same way.

### Performance Characteristics
Binary Search Trees offer lookup, insertion, and deletion in O(log n) time
while consuming O(n) space. Their performance makes them an attractive option
for certain classes of problems. For example, some sorting and search
algorithms make use of BSTs. It is important to point out however, that in the
worst case, (remember the linked list?) it has a runtime complexity of O(n) for
all operations.

While these runtime guarantees certainly do not look as good as our hash
table's, there are ways of improving upon them. For instance, if our tree is
perfectly balanced–that is, its left and right subtrees are the same height–
then the performance actually improves: lookup becomes O(log n) in the worst
case (quite an improvement over the linked list)! Such trees are known as
[balanced trees][4], and will be the topic of a future tutorial.

### Implementation Details
We will build our trees from a single `Node` type. All the relations between
nodes can be obtained from a single root. Drawing on our
[linked list implementation][1], let us define a `Node` type that satisfies an
interface of having a right and left child `Node`s as well as containing key
and value fields:

```clojure
(definterface INode
  (getLeft [])
  (getRight []))

(deftype Node
  [key
   val
   ^:volatile-mutable ^INode left
   ^:volatile-mutable ^INode right]

  INode
  (getLeft [_] left)
  (getRight [_] right))
```

Here we define a simple interface for our `Node` type. Note that our key and
value fields are immutable for now. This is primarily to simplify our
implementation.

First let us implement an insertion method for our nodes. But before we do, we
should consider our definition of the  Binary Search Tree. Specifically we want
to ensure insertions are done in such a way that the left and right sides of
our trees contain the smallest to the largest values, in that order. Why is
this important? As we will see later, the algorithms and data structures built
from Binary Search Trees rely on this ordering guarantee.

The implications for our insertion method should be fairly straightforward: we
simply want to make sure the value we insert into our tree falls to the left or
right of a given node depending on whether it is smaller or larger than the
root. This sounds like a good candidate for a recursive implementation:

```clojure
;; comparator helpers
(def gt? (comp pos? compare))

(def lt? (comp neg? compare))

(definterface INode
  ...
  (insert [k v]))

(deftype Node
  ...
  (insert [this k v]
    ;; establish a new node for insertion
    (let [n (Node. k v nil nil)]
      (cond
        ;; inserted key `k` is larger than root node's key
        (gt? k key) (if right             ;; if a right node
                      (.insert right k v) ;; recurse, else
                      (set! right n))     ;; set right to `n`

        ;; the inserted key `k` is less than the root node's key
        (lt? k key) (if left
                      (.insert left k v)
                      (set! left n))))))
```

Now we have a way to add nodes to our tree. This is a little dense so we will
take some time now to explain what is happening here. Because we know we will
be inserting a new node, the first thing we do is construct a node and bind it
in the local method scope to `n`. After that we ask if the key we would like to
insert is greater than the key of the root node. If this condition
holds we then ask if we have a right node at all. If we do, we call `insert`
again, but this time we use the `right` node as the root. If we have no right
node we simply set the right node to `n`. We repeat a similar process in the
second `cond` case, but with the left node.

Another thing to note here is our definition of the helper functions, `gt?` and
`lt?`. If our keys were limited in type to integers, we could instead simply
use Clojure's builtins (`>`, `<`) to handle these checks. But because we want
to generalize our tree to handle keys of potentially any type, we use
[comparators][5].

With this we can grow our tree. But in order for it to be useful, we also need
a way to find nodes in the tree. To do this we should define a `lookup` method.
Again we can use recursion to discover nodes that match a given search key:

```clojure
(definterface INode
  ...
  (lookup [k]))

(deftype Node
  ...
  (lookup [this k]
    ;; check if current root's key matches search key `k`
    (if (= k key)
      val
      (cond
        ;; if both a non-nil right and `k` is greater than key
        (and (gt? k key) right) (.lookup right k)

        ;; if both a non-nil left and `k` is less than key
        (and (lt? k key) left) (.lookup left k)))))
```

At this point we have a basic, working implementation of a binary search tree.
For convenience we can write a simple helper method to bootstrap tree creation:

```clojure
(defn bst [& [k v]] (Node. k v nil nil))
```

And use it to test our implementation in the REPL:

```clojure
=> (def tree (bst :foo :bar))
#'user/tree
=> (.insert tree :baz :qux)
#<Node user.Node@2443906f>
=> (.lookup tree :foo)
:bar
```

Cool, it works! With our base implementation out of the way, let us move on.

## Tree-walking

An important aspect of trees is traversal. This is because many of the benefits
of trees are derived from their physical structure and the primary way we
exploit that is by walking them in particular ways. Because trees are
generally non-linear data structures (in the case of having only one edge, we
could consider them linear) there are multiple ways to walk through its nodes.
These methods are either [depth-first][6] or [breadth-first][7] searches. In
the case of the Binary Search Tree, we will look at a specific kind of
traversal known as [in-order][8], which is considered a depth-first search.

### In-Order Traversal
This method of tree traversal visits the nodes in our tree in order, exactly as
the name might lead you to believe. For instance, if we have a simple tree of
three nodes, we first visit the left node from the root, record its value, then
we visit the root node, record its value, and finally visit the right node and
record its value. In this way we visit the nodes "in order". An interesting
property of this search is we will return a sorted view of the tree's values.

<p align="center">
    <img src="/images/in-order-traversal.png" title="The route of an in-order traversal">
</p>

Let us consider how we might implement this: in our example case, we use
a simplified tree of three nodes. But if we consider our previous
implementations of `insertion` and `lookup`, our algorithms operated on a root
and its left and right nodes. This was the foundation for the recursive
solutions there. Perhaps we can do the same here. In fact if we write a method
that sandwiches the visitation of the left and right nodes between the
visitation and value recording of the root, we will have an implementation of
in-order search!

```clojure
(definterface INode
  ...
  (inOrder []))

(deftype Node
  ...
  (inOrder [_]
    (lazy-cat
      ;; if there is a left, call inOrder with it as the root
      (when left
        (.inOrder left))

      ;; wrap the root's value with a vector
      (vector val)

      ;; if there is a right, call inOrder with it as the root
      (when right
        (.inOrder right)))))
```

Walking through this method and using our simplified tree of three nodes as an
example, we begin by checking if we have a left node. We do, so we want to
call `inOrder` on it immediately. This is because we want the left-most nodes
to appear first. This goal requires we walk to the bottom of the tree, finding
the left-most leaf, and return this node's `val` as
the first value. Moving on, as we enter the recursive call, using `left` as our
new root node, we fail to pass the `when left` conditional. This means we call
`vector` over `val`, i.e. the `val` of what was the left node of our
original tree. We then check for the presence of a right node, but since this
is a leaf node, there of course is none. The final step lazily concatenates any
values we ended up with, here only the vector of the value.

Now we have finished the first recursive call and are back in the scope of the
initial call to `inOrder`. We have our original root node and we retrieve its
value just as we did previously. Remember that the value of the `when left`
conditional is a vector containing the value of the left node. This is
important because at the end of each call we concatenate these vectors
together, and ultimately end up with a vector of all the values contained in
our tree.

Finally we move on to the `when right` conditional, which from our original
root evaluates truthy. So we call `inOrder` recursively again, doing exactly
as we did with the left node, but operating with the right node
as our root this time. Again this will yield a vector containing the value of
the right node. At last the outermost `concat` is called, merging the vectors
together and returning: `'(A B C)`.

By operating on subtrees–that is, treating portions of our larger tree as
trees in and of themselves (which they are!) we are able to reduce the problem
of traversal to a simple recursive definition. In general, this is a powerful
technique for manipulating trees.

## Pruning
Sometimes trees grow beyond their allotted space in the world. In such
situations we deploy an arborist to practice their craft. As data structure
implementers, we are tasked with this responsibility. Currently our tree is
free to grow as large as we tell it to. But it would probably be nice if we had
some process for deleting nodes we are no longer interested in.

To do this we will have to write a method that deletes a target node. Before we
get started, we should think about how this will
work. Whenever we want to delete a node the first step will be to find the node
in our tree. Once we have identified the target node, we then need to remove
it without disturbing the binary search property of the remaining tree.

1. The simple case, where our target node is a leaf, with no children. Here
we can simply remove this node by setting its parent to point to `nil`.

2. The slightly more complicated case, where we have a node with exactly one
child. Here we can move this child into the position of the target node.

3. The most complicated case, where the target node has two children. Here we
will use the structure of our tree to identify a node that would be easier to
delete, i.e. a node with one or fewer children which can be moved into the
position of the target node. What node might fit this constraint? Actually,
there are always two such nodes in a binary search tree: either the
predecessor or successor of the target node. These are the largest node in the
left subtree and the smallest node in the right subtree, respectively.

We will also have to change some aspects for our `Node` implementation.
In particular, the nodes will have to be fully mutable to accommodate deletion
and specifically deletion in the most complex case, where we actually will swap
the value of two nodes in our tree.

Let us start by updating our node's interface:

```clojure
(definterface INode
  (getLeft [])
  (getRight [])
  (setLeft [n])
  (setRight [n])
  (getKey [])
  (setKey [k])
  (getVal [])
  (setVal [v])
  (insert [k v])
  (lookup [k])
  (delete [k])
  (delete [k n])
  (inOrder []))
```

With that out of the way, the first thing we have to do is add a few helper
methods, for accessing and updating our keys and values. Then it is on to the
meat of the implementation:

```clojure
(deftype Node
  [^:volatile-mutable key
   ^:volatile-mutable val
   ^:volatile-mutable ^INode left
   ^:volatile-mutable ^INode right]

  INode
  ...
  (getKey [_] key)

  (setKey [_ k] (set! key k))

  (getVal [_] val)

  (setVal [_ v] (set! val v))

  ...

  (delete [this k]
    (.delete this k nil))

  (delete [this k parent]
    (letfn [;; a closure to help us set nodes on the parent node
            (set-on-parent [n]
              (if (identical? (.getLeft parent) this)
                (.setLeft parent n)
                (.setRight parent n)))

            ;; a function that finds the largest node in the
            ;; left subtree
            (largest [n]
              (let [right (.getRight n)]
                (when (.getRight right)
                  (largest right))
                right))]

      ;; if we have the target key, we fall into one of three
      ;; conditions
      (if (= k key)
        ;; note that the cond ordering is to ensure that we do
        ;; not match cases such as (or left right) before we
        ;; check (and left right)
        (cond
          ;; 3. two children, the most complex case: here we
          ;;    want to find either the in-order predecessor or
          ;;    successor node and replace the deleted node's
          ;;    value with its value, then clean it up
          (and left right) (let [pred (largest (.getLeft this))]
                             ;; replace the target deletion node
                             ;; with its predecessor
                             (.setKey this (.getKey pred))
                             (.setVal this (.getVal pred))

                             ;; set the deletion key on the
                             ;; predecessor and delete it as a
                             ;; simpler case
                             (.setKey pred k)
                             (.delete this k))

          ;; 1. no children, so we can simply remove the node
          (and (not left) (not right)) (set-on-parent nil)

          ;; 2. one child, so we can simply replace the old node
          ;;    with it
          :else (set-on-parent (or left right)))

        ;; otherwise we recurse, much like `lookup`
        (cond
          ;; if we have both a non-nil right node and `k` is
          ;; greater than key
          (and (gt? k key) right) (.delete right k this)

          ;; if we have both a non-nil left node and `k` is less
          ;; than key
          (and (lt? k key) left) (.delete left k this))))))
```

The primary complication of deletion is the case of a node up for deletion
which has two children. Otherwise our deletion method works much like our
lookup method, searching through the tree recursively. When we do find a node
with the key we wish to delete, we fall into a conditional block. Again, the
simple cases are a node with no children or one child. These simply replace the
target node by instructing its parent to point to the target node's child or
`nil`. There is one slight complication here: we have to identify which of the
parent's children our target node is, for this we write a small helper function
called `set-on-parent`.

In the most complex case, where we have two children, we first identify the
predecessor node. This node's key and value fields are going to be swapped with
the target node's. By doing so, we can then again call delete and fall into
a simpler case. Consider that the leftmost maximum node will always be
correctly placed when swapping with the target node, because by definition it
will always proceed all nodes in the left subtree.

Deletion is less elegant than insertion and lookup, but it still follows
a familiar recursive paradigm.

## Conclusion
This concludes our exploration of trees. But we have barely scratched the
surface of a deep and important topic in computer science. Trees and in
particular Binary Search Trees are fundamental components of many algorithms and
are certainly pervasive throughout the world of programming.

Our implementation of the BST is rudimentary but still illustrative of the
fundamentals. It is important to note that our methods for adding and finding
nodes in our implementation are recursive. Recursion is a natural method for
dealing with trees, this is because trees themselves can be thought of as
recursive.

An interesting property of our BST fell out of the in-order traversal: our
nodes ended up being returned back to us in sorted order! While this is the
only traversal method we explored in this implementation, there are others: but
practically, the only useful one where a BST is concerned is in-order,
precisely because we end up with a sorted representation of nodes.

Finally we tackled the slightly-tricky deletion algorithm. This is perhaps the
least elegant aspect of our implementation and it is important to point out
that there are other ways of representing our tree that might simplify
operations such as deletion. However for the purpose of this tutorial, we
explored an algorithm which handled deletions of nodes with two children by
identifying a predecessor node and swapping it with the node-to-be deleted.

Our trees are not balanced trees. This implies certain performance
degradations, depending upon how data is inserted into our tree. In the
pathological case performance is reduced to what we saw with linked lists. But
there are ways of mitigating this. One such strategy is called a
[Red-Black Tree][9]. This will be the topic of a future post.


[0]: http://macromancy.com/2014/01/16/data-structures-clojure-singly-linked-list.html "Singly-Linked Lists"
[1]: http://macromancy.com/2014/02/03/data-structures-clojure-hash-tables.html "Hash Tables"
[2]: http://hypirion.com/musings/understanding-persistent-vector-pt-1 "Understanding Clojure's Persistent Vectors, pt. 1"
[3]: http://en.wikipedia.org/wiki/Binary_search_tree "Binary Search Tree"
[4]: http://en.wikipedia.org/wiki/Balanced_trees "Self-balancing binary search tree"
[5]: http://docs.oracle.com/javase/7/docs/api/java/util/Comparator.html "Comparators"
[6]: http://en.wikipedia.org/wiki/Depth-first_search "Depth-first search"
[7]: http://en.wikipedia.org/wiki/Breadth-first_search "Breadth-first search"
[8]: http://en.wikipedia.org/wiki/In-order_traversal#In-order_.28symmetric.29 "In-order traversal"
[9]: http://en.wikipedia.org/wiki/Red-black_tree "Red-black tree"
