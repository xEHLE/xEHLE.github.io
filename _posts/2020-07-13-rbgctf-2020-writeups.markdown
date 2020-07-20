---
layout: post
title: "H1-2006 2020 writeup"
categories: ctf
---

## Introduction {#intro}

{% highlight go %}
package main

import (
	"bufio"
	"fmt"
	"log"
	"os"
	"strconv"
	"strings"

	"github.com/RyanCarrier/dijkstra"
)

func main() {
	// initialize the map and 200,000 numbered points
	graph := dijkstra.NewGraph()

	for i := 1; i < 200001; i++ {
		graph.AddVertex(i)
	}

	// open file with path data
	file, _ := os.Open("map.txt")
	scanner := bufio.NewScanner(file)
	for scanner.Scan() {
		x := scanner.Text()
		y := strings.Split(x, " ")
		// convert strings to int
		z1, _ := strconv.Atoi(y[0])
		z2, _ := strconv.Atoi(y[1])
		z3, _ := strconv.Atoi(y[2])
		// routes needs to be added both ways
		graph.AddArc(z1, z2, int64(z3))
		graph.AddArc(z2, z1, int64(z3))
	}

	// find and print the best path
	best, err := graph.Shortest(1, 200000)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(best.Path)

}

{% endhighlight %}

----------------

## Typeracer {#typeracer}
{: style="text-align: center"}
####  Category: Web | Solves: 184 | Points: 119
{: style="text-align: center"}

----------------

----------------

{:refdef: style="text-align: center;"}
[--- Back to Top ---](#intro)
{: refdef}

----------------