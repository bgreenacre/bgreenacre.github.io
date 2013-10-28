---
title: Something To Ponder
layout: detail
author:
    name: Brian Greenacre
    url: http://bgreenacre.github.io
---

Table conditions holds only a primary key and varchar that gives the condition a name (ie. customer_state). Table condition_values holds a primary key, a key that connects with a variable row and a value (ie. Illinois). The variable table held variable names and the values table had a key to the variable it belongs to but also a key to a condition_value (also a column that held the variable value). I hope this is making sense so far.

**SQL**

{% highlight sql %}
CREATE TABLE `variables` (
  `id` int(10) NOT NULL auto_increment,
  `name` varchar(255) NOT NULL,
  PRIMARY KEY  (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

CREATE TABLE `values` (
  `id` int(11) unsigned NOT NULL auto_increment,
  `variable_id` int(11) NOT NULL,
  `value` varchar(255) NOT NULL,
  PRIMARY KEY  (`id`),
  KEY `variable_id` (`variable_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

CREATE TABLE `condition_values` (
  `id` int(11) unsigned NOT NULL auto_increment,
  `condition_value_id` int(10) NOT NULL,
  `condition_id` int(10) NOT NULL,
  `value` varchar(255) NOT NULL,
  PRIMARY KEY  (`id`),
  KEY `condition_value_id` (`condition_value_id`),
  KEY `condition_id` (`condition_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

CREATE TABLE `conditions` (
  `id` int(11) unsigned NOT NULL auto_increment,
  `name` varchar(255) NOT NULL,
  PRIMARY KEY  (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
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

Contents of <code>getVariables()</code> function would be the following.

{% highlight php %}
$cases = array('case1' => '111', 'case2' => '112');

$selectStr = 'SELECT DISTINCT v.id, v.value, var.name, ';
$fromStr   = 'FROM `values` AS v, variables AS var';
$whereStr  = 'WHERE var.id = v.variable_id';
$superWhere = ' WHERE ';
$count = 1;

foreach ($cases as $key => $value)
{
    $superWhere .= sprintf('built.%s = "%s" AND ', $key, $value);

    $aliasName = 'conditional' . $count;

    $selectStr .= sprintf('%s.value AS %s, ', $aliasName, $key);
    $fromStr   .= sprintf(', `condition_values` AS %s, `conditions` AS c%d ', $aliasName, $count);

    if ($count === 1)
    {
        $whereStr .= sprintf(' AND %s.condition_value_id = v.id', $aliasName);
    }
    else
    {
        $whereStr .= sprintf(' AND %s.condition_value_id = conditional%d.condition_value_id', $aliasName, $count - 1);
    }

    $whereStr .= sprintf(' AND %s.condition_id = c%d.id AND c%d.name = "%s"', $aliasName, $count, $count, $key);

    ++$count;
}

$selectStr = rtrim($selectStr, ', ') . ' ';
$qStr = $selectStr . $fromStr . $whereStr;

$bigQuery = sprintf('SELECT built.name, built.value FROM (%s) AS built, %s', $qStr, rtrim($superWhere, ' AND '));

echo $bigQuery;
{% endhighlight %}