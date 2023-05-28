---
layout: post
title: "Pygame + Pathfinding"
date: 2014-04-16 12:00:00
categories: [python]
permalink: :year-:month-:day-:title
---

I swear I actually work on my side projects...

After getting up to speed with Python and [Pygame](http://pygame.org/news.html), I began to waver on the next steps towards making a game. I've attempted several times to walk the path of the game developer before, and each time I tend to run into a very interesting wall. I tend to have too much fun (and therefore spend too much time) fiddling with the inner workings of various data structures or with interconnecting various components of a rudimentary engine. For better or worse,
I've come to accept that I may not be able to focus on a single project long enough to make a functioning game.

So, instead of charging full steam ahead on getting some sort of gameplay into the project, I decided to visit an algorithm that I haven't thought about in several years: the [A\* Pathfinding Algorithm](http://en.wikipedia.org/wiki/A*_search_algorithm).

### The Path Manager

This class forms the entry point for a request for a path. This implementation of A\* will be fairly simple. Instead of keeping track of references to node objects, Python will allow us to maintain our global "map" as a series of tuples representing x and y positions. These immutable lists of values will furthermore be able to be used as keys into dictionaries, allowing for fast lookups should particular nodes require additional data.

```python
import math

class PathManager:
    def __init__(self, w=0, h=0 obstacles=[]):
        self.w = w
        self.h = h
        self.obstacles = obstacles
        self.path_map = self.construct_map()

    def reset_obstacles(self, obstacles=[]):
        self.obstacles = obstacles
        self.path_map = self.construct_map()
    
    def construct_map(self):
        path_map = PathMap(self.w, self.h)
        path_map.update_blocked_nodes(self.obstacles)
        return path_map

    @staticmethod
    def dist_heuristic(from_node, to_node):
        dx = int(math.fabs(from_node[0] - to_node[0]))
        dy = int(math.fabs(from_node[1] - to_node[1]))
        return dx ** 2 + dy ** 2

    @staticmethod
    def dist(from_node, to_node):
        dx = int(math.fabs(from_node[0] - to_node[0]))
        dy = int(math.fabs(from_node[1] - to_node[1]))
        return dx if dx > dy else dy

    def generate_path(self, from_node, to_node):
        pass

```

The constructor for a PathManager takes as input the width and height of the world, and an optional list of 'obstacle' nodes. These nodes will be considered to be impassable by the A\* algorithm.

The *dist\_heuristic* and *dist* methods will be used to determine the best possible selection for steps along our shortest path. This is the part of the system which seems to be more art than science. Even the slightest tweaks in the type of heuristic used can have a tremendous effect on the speed and accuracy of the overall determination of a path. I found that using the Euclidean distance for the heuristic (squared, to avoid the relatively-costly square root operation) provided a
nice balance of efficiency and accuracy.

For the distance method itself, I had originally been simply using the Manhattan distance between two nodes (that is, the number of nodes between two points if diagonal movements are not allowed-- think of the fastest way of arriving at a location if the moving entity is the castle piece on a chess board). This worked fine, but felt a little unrealistic when the movements stretched across the map. The current implementation allows for diagonal movements, but doesn't treat them as being
any faster or slower than a non-diagonal step.

Also notice that suspiciously empty *generate\_path* method... we'll get back to that.

### The Map

As mentioned, our 'map' is really just a collection of *(x, y)* coordinates that will be used as dictionary keys. Furthermore, in the actual persistence of a map, we only really care about points that are actually of interest to the A\* algorithm. In our case, the set of 'obstacle' nodes is really all we need. This allows us to maintain a very lightweight map. Once we dive into the search algorithm itself, we will be constantly checking for possible steps to take from any given
context node. We can move the burden of this calculation to the map itself:

```python
class PathMap:
    def __init__(self, w, h):
        self.w = w
        self.h = h
        self.blocked_nodes = set()

    def update_blocked_nodes(self, nodes):
        self.blocked_nodes = set()
        for node in nodes:
            self.blocked_nodes.add(node)

    def get_neighbor_nodes(self, node, step=1):
        neighbor_nodes = []
        for y in range(-1, 2):
            for x in range(-1, 2):
                if x == y == 0:
                    continue
                neighbor_node = (node[0] + x * step, node[1] + y * step)
                if neighbor_node in self.blocked_nodes:
                    continue
                else:
                    neighbor_nodes.append(neighbor_node)
        return neighbor_nodes
```

The *get\_neighbor\_nodes* method will take as input a node (coordinate tuple), and an optional step value, and will return an array of all possible nodes that are valid for consideration. Note that currently this makes no checks against the size of the world.

### The Path

Now that we've got a storage mechanism squared away, we need a class to help out with the actual determination of the shortest path. I've omitted from the code below a number of getter and setter methods for brevity, but hopefully the core of the class remains:

```python
class Path:
    def __init__(self):
        self.open_set = set()
        self.closed_set = set()
        self.g = {}
        self.f = {}
        self.parent = {}
        self.path = []

    def construct(self, tile=None):
        if not tile:
            return self.path
        self.path.append(tile)
        parent = self.get_parent(tile)
        while parent:
            self.path.append(parent)
            parent = self.get_parent(parent)
        self.path.reverse()
        return self

        def first_open_node(self):
            return sorted(self.open_set, 
                          key=lambda x: self.f[x] if x in self.f else float("inf"))[0]
    
        def close_enough(self, from_node, to_node):
            dx = int(from_node[0] - to_node[0])
            dy = int(from_node[1] - to_node[1])
            return (dx ** 2 + dy ** 2) <= 100
```

The Path object will take care not only of keeping track of *g* and *f* values for the nodes themselves (values which are used to determine the appropriate steps to take in a path) but also the 'parent' nodes of each step. Once a node is determined to be close enough to the destination location (via *close\_enough*) it is used as the starting point for the construct method, which reverses this chain of nodes until the starting node is retrieved, returning the final path as an array of
coordinates.

Note that our search algorithm will constantly be looking for the 'first node in the open set,' which really means the node in the set of nodes not yet considered which has the lowest *f* value. The method *first\_open\_node* will return exactly that node.

### The Algorithm

With the framework ready to go, it's time to revisit the *generate\_path* method in the PathManager class:

```python
    def generate_path(self, from_node, to_node):
        path = Path()
        path.add_open_set(from_node)
        path.set_g(from_node, 0)
        if to_node in self.path_map.blocked_nodes:
            return path.construct(from_node)
        while len(path.open_set) > 0:
            context_node = path.first_open_node()
            path.remove_open_set(context_node)
            path.add_closed_set(context_node)
            for neighbor_node in self.path_map.get_neighbor_nodes(context_node, step=5):
                if path.contains_closed_set(neighbor_node):
                    continue
                if path.close_enough(neighbor_node, to_node):
                    path.set_parent(neighbor_node, context_node)
                    return path.construct(neighbor_node)
                temp_cost = path.get_g(context_node) + PathManager.dist(context_node, neighbor_node)
                if temp_cost < path.get_g(neighbor_node):
                    path.add_open_set(neighbor_node)
                    path.set_parent(neighbor_node, context_node)
                    path.set_g(neighbor_node, temp_cost)
                    path.set_f(neighbor_node, path.get_g(neighbor_node) + \
                               PathManager.dist_heuristic(neighbor_node, to_node))
        return path.construct(from_node)
```

The implemented code follows the wikipedia algorithm fairly closely. The two parameters, *from\_node* and *to\_node*, are just *(x, y)* tuple coordinates, just as the other classes expect. As an initial step, the *g* value (the calculated cost of traveling from the *start\_node* to this node) of the *start\_node* is set to 0. This node is then added to the *open\_set* (the set of nodes to be considered as part of the shortest path to the *to\_node*).

If the *to\_node* is determined to be invalid (in our case this means it is part of the 'obstacle' nodes passed into the PathManager at the very beginning) then the method returns an empty path. If the *to\_node* is valid, then the meat of the algorithm begins.

The node in the *open\_set* with the minimum g value is retrieved and is considered to be the '*context\_node*'. This node is removed from the *open\_set* and placed into the closed set (preventing the algorithm from falling into a loop) and the map is queried to determine all valid choices of travel from the *context\_node*. If a neighbor node is found to be close enough to the *to\_node* to be considered the end of the path, the algorithm completes and the path is constructed. Otherwise,
each neighbor node is given a new *g* value and an associated *f* value (the estimated cost of traveling from this node to the *to\_node*). Finally, if the new *g* value is smaller than the old *g* value for this node, the new *g* value is saved, and the path up to this point is updated to take into account this shorter route.

### The Result

I added a bit of code to draw the obstacles themselves, in addition to highlighting some of the decision making the A\* algorithm is doing. The green line is the calculated shortest path. Green dots were steps that were considered to be possibly involved in the shortest path, and red dots are steps that were considered to definitely not be part of the shortest path.

![Pathfinding Dude]({{ site.url }}/assets/2014-04-16-pygame-pathfinding/pathfinding.gif)

[Full project](https://github.com/drewmalin/pygame_pathfinding)