# TODO

- Cluster size and expected running time info
- Update all paths in README and in scripts from jpacker-dev to production ready mortar-examples
- Rename this repo "mortar-pagerank"
- QA all the things

# Welcome to Mortar!

Mortar is a platform-as-a-service for Hadoop.  With Mortar, you can run jobs on Hadoop using Apache Pig and Python without any special training.  You create your project using the Mortar Development Framework, deploy code using the Git revision control system, and Mortar does the rest.

# Getting Started

This Mortar project is a generic framework for calculating Pageranks for the nodes of any directed graph. There are two example scripts which demonstrate its use. `twitter-pagerank` runs Pagerank on a graph of Twitter follower relationships to find the most influential Twitter users. `patents-pagerank` runs Pagerank on a graph of patent citation relationships to find which organizations produce the most influential patents. Alternatively, you can configure the template script `my-pagerank` to run Pagerank on your own data (see the last section of this README). Regardless of which you choose, there are some steps to get started:

1. [Signup for a Mortar account](https://app.mortardata.com/signup)
1. [Install the Mortar Development Framework](http://help.mortardata.com/#!/install_mortar_development_framework)
1. Clone this repository to your computer and register it as a project with Mortar:

        git clone git@github.com:mortardata/mortar-pagerank.git
        cd mortar-pagerank
        mortar register mortar-pagerank

## Twitter Example

Once everything is set up you can run the Twitter example by doing:

        mortar run twitter-pagerank --clustersize 5

This script runs on the a graph of follower relationships between the 100k Twitter users with the most overall followers (as in, only relationships between users who are already known to be at least modestly influential are considered). The data comes from [What is Twitter, a Social Network or a News Media?](http://an.kaist.ac.kr/traces/WWW2010.html) and was generated in early 2010. The script will finish in about SOME TIME on an N-node cluster.

## Patents Example

You can run the Patents example by doing:

        mortar run patents-pagerank --clustersize 5

This script runs on a graph of patent citations for US patents granted between 2007 and 2012. As the citation graph itself is very sparse, we reduce the graph so that nodes are organizations instead of individual patents: the edge between organization A and organization B has a weight equal to the number of patents filed by organization A which cite patents filed by organization B. The resulting reduced graph is actually quite small due to the short timeframe of the data: unfortunately, the USPTO seems to change their data format every few years, and we only had time to parse the most recent of these formats.

The patent data we're using is generated from public information made available in XML form by the United States Patent Office (USPTO) and hosted by Google [here](http://www.google.com/googlebooks/uspto-patents-grants-biblio.html). We parsed the XML into PigStorage format at put it up at S3 LOCATION. There is actually a lot of information beyond just the citations which we don't use in this project--see the LOAD\_PATENTS macro in `./macros/patents.pig` to see a schema of the available information.

The script will finish in about SOME TIME on an N-node cluster.

# The Pagerank Algorithm

Pagerank simulates a random walk over a weighted directed graph, where the probability of going from a node N to a node M over an edge is that edge's weight divided by the sum of the outgoing edge weights for node N (unweighted graphs simply set each edge weight to 1.0). The pagerank of a node is the probability that after a large number of steps (starting at a random node) the walk will end up at that node. There is also a chance at each step that the walk will "teleport" to a completely random node: this added factor allows the algorithm to function even if there are "attractors" (nodes with no outgoing edges) which would otherwise trap the walk.

Pagerank is an iterative algorithm.  Each pass through the algorithm relies on the previous pass' output pageranks (or in the case of the first pass a set of default pageranks generated for each node). The algorithm is considered done when a new pass through the data produces results that are "close enough" to the previous pass. See http://en.wikipedia.org/wiki/Pagerank for a more detailed algorithm explanation.

# What's inside

## Control Scripts

The files `./controlscripts/twitter-pagerank.py` and `./controlscripts/patents-pagerank.py` are the top level scripts that we run in Mortar to find Pageranks, for the Twitter graph and the Patent Citations graph respectively.  Using [Embedded Pig](http://help.mortardata.com/reference/pig/embedded_pig), this Jython code is responsible for running our various pig scripts in the correct order and with the correct parameters.

The file `./controlscripts/my-pagerank.py` is the template from which the pre-made controlscripts were developed from, and is a good starting point for configuring Pagerank to run on your own data.

For easier debugging of control scripts all print statements are included in the pig logs shown on the job details page in the Mortar web application.

## Pig Scripts

This project contains seven pig scripts: three for generating graphs to use with Pagerank, two which implement the Pagerank algorithm, and two which optionally postprocess Pagerank output in different ways.

### generate\_twitter\_graph.pig

This pig script takes the full Twitter follower graph from 2010 and returns the subset of the graph that includes only the top 100 000 users.

### generate\_patent\_citation\_graph.pig

This pig script takes the US Patent Grant dataset (already in PigStorage form), projects from all the available information just the citation graph, and does the reduction so that each node is an organization instead of an individual patent, as described earlier. It also makes a list of the most-cited organizations: we use this as a 

### generate\_my\_graph.pig

This template script shows steps you might take to generate a graph from your own data to input into the Pagerank scripts.

### pagerank\_preprocess.pig

This pig script takes an input graph and converts it into the format that we'll use for running the iterative pagerank algorithm. This script is also responsible for setting the starting pagerank values for each node.

### pagerank\_iterate.pig

This pig script calculates updated pagerank values for each node in the graph.  It takes as input the previous pagerank values calculated for each node.  This script also calculates a 'max\_diff' value that is the largest change in pagerank for any node in the graph.  This value is used by the control script to determine if its worth running another iteration to calculate even more accurate pagerank values.

### pagerank\_postprocess\_with\_name\_join.pig

This pig script takes the final iteration output of nodes and their pageranks and joins it to an index mapping each node ID to a human-readable name. It then outputs the TOP_N nodes by pagerank to S3.

### pagerank\_postprocess\_without\_name\_join.pig

This pig script takes the final iteration output of nodes and their pageranks and outputs the TOP_N nodes to S3. It _does not_ join to any name index. This is for nodes which have human-readable identifiers already.

# Pagerank Parameters

## Damping Factor

The damping factor (in academic papers, usually the greek letter alpha) determines the variance of the final output pageranks.  This is a number between 0 and 1 where (1 - DAMPING\_FACTOR) is the probability of the random walk teleporting to a random node in the graph. At 0 every node would have the same pagerank (since edges would never be followed).  Setting it to 1 would mean the walks get trapped by attractor nodes and would rarely visit nodes with no incoming edges.

A common value for the damping factor is 0.85. Lowering the damping factor will exponentially speed up convergence, but at the cost of your pageranks not reflecting as much of the true underlying structure of the graph.

## Convergence Threshold

The intermediate pageranks calculated by each iteration in the algorithm will converge to fixed values for each node after an infinite number of iterations. As for many use cases this is too long to wait, we stop the iterations when the latest iteration's results are "close enough" to the last iteration's results as determined by some convergence metric.

We make our convergence metric the sum of all of the "ordering-rank" changes of the pageranks. "ordering-rank" in this case means the pagerank's position in the ordering of all pageranks from greatest to least--so the highest pagerank has ordering-rank 1, and the least has ordering-rank N. A swap of ranks 2 and 5 would be two ordering-rank changes of magnitude |2-5| = 3, so it would add 2*3 = 6 to the sum.

We converge when this sum of the ordering-rank changes from one iteration to the next is less than 0.00005 * (# nodes)^2. The squared term is because the maximum possible number of ordering-rank changes for N nodes is proportional to N^2. The constant term is arbitrarily chosen, so your data might work best with a different value. We found it to be appropriate for both the larger Twitter graph and the small patent graph.

## Maximum Number of Iterations

In case a graph takes too long to converge, the pagerank script will always halt after a number of iterations equal to the parameter MAX\_NUM\_ITERATIONS.

# Using your own data

## Generate a graph

To run Pagerank on your own data, you must first generate a directed graph from your data. The pigscript `my_preprocessing_script.pig` has some code snippets to help you get started. Your output should have a schema:

    from, to, weight

`from` and `to` are the ids of the nodes the edge comes from and goes to respectively. They can be of type int, long, chararray, or bytearray. `weight` is a measure of the magnitude or significance of an edge. It can be of type int, long, float, or double. Your output can have any delimiter (you will specify your choice in the controlscript); tabs are default for PigStorage.

If your graph is undirected, you must make two records for every connection, one from A to B and another from B to A. If your graph is unweighted, simply make every weight 1.0.

## Configuring the controlscript

Next, edit the controlscript `my-pagerank.py` to use your data. Each parameter that needs to be set is listed and explained in the template.

## Run your controlscript

Finally, to run Pagerank on your data, do:

    mortar run my-pagerank --cluster-size N

where N is an appropriate number for the size of your data.