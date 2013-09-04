---
title: Something To Ponder
layout: detail
author:
    name: Brian Greenacre
    url: http://bgreenacre.github.io
---

My first professional web development job was at a small direct marketing firm. My job was essentially maintenance and enhancements on this poorly developed/planned application that essentially generated pdfs which would then be sent off to printers. After about 2 months of working there the owner decided to invest into building a new application to essentially do the same thing plus several new features. Great! Unfortunately I was a junior dev then and I just wasn't capable of leading such a project. So they hired two developers one of which was named Matt.

Matt was actually a year younger than me but he clearly had more experience than I did. We spent quite a bit of time talking about the new project so even though I wasn't working on it directly I had plenty of input as to how it was built. One particular aspect was how the variables and values of those variables were stored in the database. The _big problem_ with the way it was currently stored and utilized is it wasn't in a database at all. It was, regretfully, inside a single _massive_ php file that was probably 10,000 lines or more.

Why?

Well, why doesn't entirely matter but I'll focus on the _technical_ reason since this post is pretty much focused around that. So for each pdf template there can be several variables that can be used to inject copy inline where the variable is within a PDF Block (think <code>str_replace(':variable_name:', 'some content should go here', $some_copy)</code>) but each variable can have one of several defined variables based on an arbitrary amount of conditions that need to be met. So guess how those conditions were tested for each variable? Yup. A ton of if/if else/else conditions were wrapping the assignment of each of these variables. WTF right? Basically, the people that wrote this file originally just didn't know how to manage these conditions nor did the client (my boss). During my time spent there I tried very hard to improve that file but due to the size, complexity, lack of testing, importance and my lack of experience I just couldn't ever get it into something manageable but I had a very good idea.

**The Idea**

At the time, I had one solution in mind and that was to get all the variables and their values into a database schema. Accomplishing that alone isn't that hard but I wanted to also store the logical conditions in which each value was assigned to a variable. That's the hard part. How does one store those conditions per variable but also how does one query for that? The first part of this question I came up with, surprisingly, quickly. My solution involved 4 tables; variables, conditions, values and condition_values. The tables variables and values are pretty self-explanitory but conditions and condition_values is the fun part.

Table conditions holds only a primary key and varchar that gives the condition a name (ie. customer_state). Table condition_values holds a primary key, a key that connects with a variable row and a value (ie. Illinois). The variable table held variable names and the values table had a key to the variable it belongs to but also a key to a condition_value (also a column that held the variable value). I hope this is making sense so far.

**SQL**

{% highlight sql %}
CREATE TABLE `variables` (
  `id` int(10) NOT NULL auto_increment,
  `name` varchar(255) NOT NULL,
  PRIMARY KEY  (`id`)
) ENGINE=MyISAM AUTO_INCREMENT=1 DEFAULT CHARSET=latin1;

CREATE TABLE `values` (
  `id` int(11) unsigned NOT NULL auto_increment,
  `condition_value_id` int(10) NOT NULL,
  `condition_id` int(10) NOT NULL,
  `value` varchar(255) NOT NULL,
  PRIMARY KEY  (`id`),
  KEY `condition_value_id` (`row_id`),
  KEY `condition_id` (`case_id`)
) ENGINE=MyISAM AUTO_INCREMENT=1 DEFAULT CHARSET=latin1;

CREATE TABLE `conditions` (
  `id` int(11) unsigned NOT NULL auto_increment,
  `name` varchar(255) NOT NULL,
  PRIMARY KEY  (`id`)
) ENGINE=MyISAM AUTO_INCREMENT=1 DEFAULT CHARSET=latin1;

CREATE TABLE `condition_values` (
  `id` int(11) unsigned NOT NULL auto_increment,
  `variable_id` int(11) NOT NULL,
  `value` varchar(255) NOT NULL,
  PRIMARY KEY  (`id`),
  KEY `variable_id` (`base_tag_id`)
) ENGINE=MyISAM AUTO_INCREMENT=1 DEFAULT CHARSET=latin1;
{% endhighlight %}

The SQL should make it pretty clear what each table does. The next problem was how to query these tables with a set of conditions to get variables and their values in, preferably, an associative array or iterable object. It also needed to be done in one query.

Something like ...
{% highlight php %}
$conditionsArr = array(
  'condition1' => 4,
  'condition2' => 1,
  'condition3' => '2|4',
);

var_dump(getVariables($conditionsArr));

// Would return an array of variables
// 
// array(
//   'var1' => 'some value',
//   'var2' => 'other value',
// );
{% endhighlight %}