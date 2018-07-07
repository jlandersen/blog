+++
author = "admin"
categories = ["abap", "C#", "java", "data intensive systems", "large data set", "reflection", "sap", "rtts"]
date = 2012-11-01T14:56:50Z
description = ""
draft = false
slug = "abap-reflection-part-one-the-rtts-and-type-introspection"
tags = ["abap", "C#", "java", "data intensive systems", "large data set", "reflection", "sap", "rtts"]
title = "ABAP Reflection - Part One (The RTTS and Type Introspection)"

+++


Reflection provides the ability to examine, modify and create data types and behaviour at runtime. Developers of C# (or Java) not familiar with this concept, have most likely encountered it at some point, or even used it, through the .NET or Java reflection library. Have you ever used the typeof operator or invoked the GetType() methods in C#? Then you have already encountered it first hand.

Recently I found the need for reflection in ABAP, all the way from type inspection to creation of new data types at runtime, which I found to be documented only very superficial. This (and the following) post is a result of my experiences working with reflection in ABAP. In this first post I will briefly introduce an overview of reflection in ABAP and how this can be used to inspect types at runtime. If you just want to get down with reflection in ABAP I suggest you skip the next section.

### Why bother and why did I need it?

Reflection provides the developer with a number of significant opportunities that one might not realise immediately. Besides useful features such as checking whether two objects are of the same type at runtime, these types can also be further inspected for members, methods and so on. With reflection it is possible to see, whether an object has a member with a certain name at runtime or modify the value of a member not known at compile time.

In my case I needed it for a specific case, for an ABAP program being developed. The problem was to develop a program that would basically provide the following features for generating reports of data in the system:

- Let the user choose freely any number of fields/columns between all SAP HCM Master Record database tables (basically all infotypes) as well as custom added database tables
- Allow the user to specify search criteria on any of the chosen fields as well as choose which fields should be part of the resulting report generated

So basically a small program that would do something equivalent to a BI solution, however, in this case this was not an option. Considering the users freedom to choose any fields on many different database tables (with up to 80 columns each), and with large data sets in each table, the two primary concerns were performance  and how queried data should be kept in-memory for processing. Processing each database table separately  and writing intermediate results to some storage was not an option as each column could require its value to undergo some processing with interdependencies between values across tables. Also the results generated for the output report needed to be displayed as a coherent timeline for each employee in the results, requiring all column values across database tables to be displayed coherently together in each row for an employee with multiple time intervals.

My solution to this – and especially on minimising the memory usage? Use reflection to create a new type at runtime – more specifically a new record data structure. This record would in this case be equivalent to a single row to be displayed in the resulting report grid and have a field for each column in a database table the user selected for display. I illustrated this below. This also added two benefits – first, being able to keep each field in memory with their respective datatype as defined in its database table and second, being able to use dynamic SQL in ABAP to only select the needed columns when querying the database – and as you might know, database access is usually the best place to start when performance is important, especially in SAP.

![ABAP Reflection Memory Usage](/images/2015/04/abapreflectionmotivationalproblem1.png)

So that was one of my small moments of need for reflection in ABAP – lets get down to business and see what it is.

### Reflection in ABAP through Runtime Type Services (RTTS)

In ABAP, reflection is provided through RunTime Type Services (RTTS). Basically this provides two main features – identifying types and descriptions at runtime, and creating types dynamically. More concretely RTTS is the combination of:

* *RunTime Type Identification (RTTI)* - type identification and description (type introspection)
* *RunTime Type Creation (RTTC)* - dynamic type creation

In this part, I will focus on RTTS in general as well as RTTI.

#### RTTS and Class Hierarchy

The RTTS is implemented as system classes that are available in all ABAP programs. There are only a few general principles to consider:

- For each type kind there exists an RTTI description class
- A concrete type is defined by its type object, and for every type, one type object exists at runtime
- As a bi-implication, each type object defines exactly one type
- Properties about a type are defined by attributes of the type object

So what this means is that each kind of type has a corresponding RTTS class (e.g. all elementary data types have one corresponding type class) and for each concrete type of that kind there will be a single object of that class at runtime (e.g. the elementary integer data type has one single type object that defines it). Lets get an overview of all of this.

All RTTS related classes are named CL\_ABAP\_\*DESCR, depending on which type kind the class represents. CL\_ABAP\_ELEMDESCR is the description class for elementary datatypes, CL\_ABAP\_TABLEDESCR is for table types and so on. Illustrated below, is the complete hierarchy of type classes (that closely resembles the ABAP type hierarchy).

![RTTS Type Hierarchy](/images/2015/04/classhierarchy.png)
RTTS Type Hierarchy

The type class hierarchy is slighty different than the [ABAP type hierarchy](http://help.sap.com/abapdocu_70/en/ABENTYPES.htm), however, with less classes than types. This is a result of only type kinds having a corresponding type class. Specific details about a type are represented by the attributes on the object of that class. E.g. in ABAP we have many different kinds of internal tables – standard tables, hashed tables, sorted tables and so on. All of these are described by the same CL\_ABAP\_TABLEDESCR class, but at runtime they will be different by each having an object of the type CL\_ABAP\_TABLEDESCR, with attributes that describe whether it is e.g. a hashed table.

As a final note on RTTS in general, I want to clear up some possible naming confusion that appears to arise in general. In the old days, only type introspection was available, and not dynamic type creation. This was named RTTI in ABAP. Later RTTC became available  and both  were synthesised into RTTS. There are, however, no specific RTTI or RTTC classes. RTTS  is available through the previously mentioned type classes. We simply divide the interface of each class into RTTI and RTTC methods and attributes. So any method on any of the CL\_ABAP\_\*DESCR classes that relate to type introspection can be categorised as an RTTI method and any method that relates to the creation of a new type object of some type kind will be categorised as an RTTC method.

#### RTTI Example

Lets get our hands a bit dirty and do some runtime examination of data types. The superclass CL\_ABAP\_TYPEDESCR, which is also the root of the RTTS type hierarchy, defines a set of (primarily RTTI related) static methods that are available for all the different type kinds. The two most used methods are DESCRIBE\_BY
\_DATA and DESCRIBE\_BY\_NAME that each returns the corresponding type object for some data object at runtime. These are used by either by providing a reference to the data object in question or the relative name of the type, as input argument respectively. Note that for a reference to some data object (e.g. a reference to an integer or table) the DESCRIBE\_BY\_DATA\_REF method must be used, and likewise for references to reference types the DESCRIBE\_BY\_OBJECT\_REF method is available.

In the following example the RTTI methods are used to get the type object for the elementary integer data type. We declare a simple integer and retrieve its type object using both methods – first by passing the data object in question and secondly by using the relative name of the integer type in ABAP.

So far so good – lets see what this type object can tell us about the integer type by inspecting the object referenced by lr\_typeobj\_for\_i (which is the same object reference assigned in both assignments):

![ABAP](/images/2015/04/RTTIIntegerInspect.png)

The elementary integer type object tells us that the kind of the type object is an integer, it has no decimals and occupies 4 bytes. Nothing extremely exciting or surprising for this type, but nonetheless, it shows us the principle of a type object.

Lets go ahead and use the DDIC to get the type object for the data structure that is underlying of the database table for the pa0002 infotype. We use the CL\_ABAP\_STRUCTDESCR for accessing type information specific to a data structure:

![ABAP SAP The PA0002 data structure type object](/images/2015/04/RTTIPA0002Inspect1.png)

The PA0002 data structure occupies 988 bytes, and the CL\_ABAP\_STRUCTDESCR specific attributes defines the structure as a flat structure. Finally the COMPONENTS attribute provides an internal table that lists all the individual fields on the structure by their name, including the data type of each of these (by their relative name). By combining the DDIC and RTTS we are able to examine a database table, and see all of its columns and the data type of each column, at runtime. Furthermore we could use this information to get the type object for each of these columns.

These examples shows just a small piece of the RTTI features. One can do many more interesting type inspections, e.g. determining which attributes or methods are available on a class or concrete object at runtime.

Finally we can also create new data objects using the type objects. The following shows the creation of a new data structure of the pa0002 type previously obtained, and assigning a value to the pernr field on this structure:

In this case we know the type and thus the field symbol used to access the individual fields of the structure created as any other structure. This, however, is not always be the case e.g. if we create a complete new data type at runtime using RTTC. In such case we would have to work with the generic programming parts of ABAP to assign a value to a field. We will get to see more advanced cases like this in the next part.

 


