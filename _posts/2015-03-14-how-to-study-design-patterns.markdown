---
layout: post
title: "How to Study Design Patterns"
date: 2015-03-14 23:23:18 -0400
comments: true
categories: craftsmanship design-patterns
---
One of the schisms in the development community is between devotees and detractors of the book [Design Patterns](http://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612/ref=sr_1_1?ie=UTF8&qid=1426390303&sr=8-1&keywords=design+patterns). For those unfamiliar with the book, it curates several patterns that arise frequently in OOD, describing when it is appropriate to use them.

As it turns out, there's a similar rift in the game of Go around [Joseki](http://en.wikipedia.org/wiki/Joseki). In [Toshiro Kageyama](http://en.wikipedia.org/wiki/Toshiro_Kageyama)'s amazing book [Lessons in the Fundamentals of Go](http://www.amazon.com/Lessons-Fundamentals-Beginner-Elementary-Books/dp/4906574289), there is a chapter titled "How to Study Joseki", which reconciles the two positions.

In this article, we will be looking at how these ideas can be applied to Design Patterns. We will also spend some time applying this approach to a pattern, to see how these ideas work in practice.

How *not* to Use Design Patterns
----------------------------------
Consider the greenhorn developer, having just read the book cover to cover. He sees a situation looking similar to the State pattern, and sets to work applying it. Let's take a look at his code:

{% codeblock lang:java %}
class EquilateralPolygon {
    private Geometry geometry;

    public EquilateralPolygon(String geometry_name, double side_length) {
	if (geometry_name.equals("square")) {
	    geometry = new Square(side_length);
	} else if (geometry_name.equals("triangle")) {
	    geometry = new Triangle(side_length);
	}
    }

    public double area() {
	return geometry.area();
    }
}

interface Geometry {
    double area();
}

class Square implements Geometry {
    private double side_length;
    public Square(double side_length) {
	this.side_length = side_length;
    }

    public double area() {
	return Math.pow(side_length, 2) * Math.sqrt(3) / 4;
    }
}

class Triangle implements Geometry {
    private double side_length;
    public Triangle(double side_length) {
	this.side_length = side_length;
    }

    public double area() {
	return Math.pow(side_length, 2);
    }
}
{% endcodeblock %}

This reveals a poor understanding of its underlying purpose. The State pattern is intended to allow the subclass of state to change throughout the life of the object, otherwise it needlessly adds complexity.  It would have been much better to do this instead:

{% codeblock lang:java %}
abstract class EquilateralPolygon {
    protected double side_length;
    public EquilateralPolygon(double side_length) {
	this.side_length = side_length;
    }

    public abstract double area();
}

class Square extends EquilateralPolygon {
    public Square(double side_length) {
	super(side_length);
    }

    public double area() {
	return Math.pow(side_length, 2) * Math.sqrt(3) / 4;
    }
}

class Triangle extends EquilateralPolygon {
    public Triangle(double side_length) {
	super(side_length);
    }

    public double area() {
	return Math.pow(side_length, 2);
    }
}
{% endcodeblock %}

Situations like this are asource of depare in experienced developers, causing the opinion that studying Design Patterns is misleading and damaging to designs. They may even mutter something about how nothing can replace experience.

The Proper Way to Study Design Patterns
=======================================
1. Don't just read the pattern, a deeper understanding is required to apply it correctly.
2. Each pattern is intended to solve a specific problem. Examine every line of an idealized implementation and contemplate the implications of every conceivable variation.
3. Consider how surrounding context can influence the needs of the pattern. A variation isn't necessarily invalid if external factors introduce additional needs.

Put another way, the 23 published Design Patterns are by no means a definitive list, and there is endless variation even within covered patterns. There are thousands of slight variations to these patterns in the world, with more being created daily. The only hope one has of putting them into practice is to have a deep understanding of their intent.

Case Study: The Memento Pattern
===============================
Let's look at the Memento Pattern to better understand this approach.  Keep in mind that this is not a complete treatment of the pattern, but a sample to give you an idea.  I encourage you to continue studying the pattern independently.

The first step in studying any pattern is reading about it. If you have the book, you should go through the relevant section now. Now let's look at an idealized version of the pattern in java:

{% codeblock lang:java %}
class Memento {}

class Originator {
    private Object state;
    public void setMemento(Memento m) {
	state = ((OriginatorMemento) m).getState();
    }

    public Memento createMemento() {
	return new OriginatorMemento(state);
    }

    private class OriginatorMemento extends Memento {
	private Object state;
	public OriginatorMemento(Object state) {
	    this.state = state;
	}

	public Object getState() {
	    return state;
	}
    }
}

class CareTaker {
    public void interact() {
	Originator o = new Originator();
	Memento m = o.createMemento();
	o.setMemento(m);
    }
}
{% endcodeblock %}

Varying Visibility
------------------
OriginatorMemento's visibility is private to Originator, which is designed hide the mechanics of Originator from the outside world. If we made the class public, how would that affect the pattern?  For one thing, it would create a way to mutate Originator outside of its official interface. This could cause problems if the implementation needs to be changed, such as for performance or adding new features.

Now let's think about the broader context, is there anything that could make us want to modify the visibility? Suppose we want to use Originator as a base class, in which subclasses have implementations which vary enough that the original state no longer makes sense. This seems like a reasonable scenario for switching the visibility to protected.

Is there a broader context in which making OriginatorMemento public is a good idea? My gut reaction is no, as a crucial piece of the pattern is the ability to hide internal implementation.  If that state can be modified outside of the Originator, we might as well do this instead:

{% codeblock lang:java %}
class Originator {
    private Object state;
    public void setState(Object s) {
	state = s;
    }

    public Object getState() {
	return state;
    }
}

class CareTaker {
    public void interact() {
	Originator o = new Originator();
	Object s = o.getState();
	o.setState(s);
    }
}
{% endcodeblock %}
