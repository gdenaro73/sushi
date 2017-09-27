# SUSHI

## About

SUSHI is an automatic test case generator for Java programs, aimed at (high) branch coverage. It leverages a technique called 
symbolic execution to calculate, from a program path that reaches a branch, a *path constraint*, i.e., a constraint on the 
program inputs that, when satisfied, allows to cover the path. SUSHI calculates path constraints by exploiting the symbolic 
executor [JBSE](http://pietrobraione.github.io/jbse/). To solve path constraints SUSHI transforms them in search problems 
and feeds by them the search-based test case generator [EvoSuite](http://www.evosuite.org/).

The main advantage of SUSHI is its ability of generating tests for programs that take as inputs complex data structures, 
allowing, for instance, to automatically test compiler passes taking parse trees as inputs.

## Building SUSHI ##

SUSHI is composed of several projects that are imported as git submodules. To get the code for all dependent project, you should clone the SUSHI git repository and then run `git submodules init && git submodule update`. 

To build SUSHI we provide a Maven script and compilation is invoked with `mvn compile`. The supported Maven goals are:

* clean
* compile
* test
* package

We advise *not* to rely on the Eclipse default mechanism to import a Maven project from a repository, but rather to clone the git repository, run `mvn eclipse:eclipse` to create the Eclipse project files, and finally import the project under Eclipse. 

## Dependencies

SUSHI depends on some software packages, most of which are already included in the repository. You will need to fix two
dependencies manually. The first dependency is [Z3](https://github.com/Z3Prover/z3). The second dependency is the [GNU Linear Programming Kit (GLPK)](https://www.gnu.org/software/glpk/) and its Java wrapper [GLPK-Java](http://glpk-java.sourceforge.net/). If your operating system is Debian or Ubuntu you can install the libglpk and libglpk-java packages. Under OSX there is a GLPK package under Macports, but no package for GLPK-Java. In the worst case you will need to install GLPK and/or GLPK-Java from sources, in which case consider that both have many dependencies on their own. Once done that you need to reconfigure the Eclipse SUSHI project so it uses the GLPK you installed. Note that the SUSHI Eclipse project contains a glpk-java.jar library, but you do *not* want to use that. So right-click the SUSHI Eclipse project in the Eclipse package explorer, and select Build Path > Configure Build Path... from the contextual menu. Then select the Libraries tab, remove the reference to glpk-java.jar, and add a new reference to the glpk-java.jar you previously installed. Finally, click the triangle on the left of the added reference, select Native Library Location, click the Edit button and enter the location of the JNI libraries produced by GLPK-Java.

## Usage

Once set the classpath you can launch SUSHI as follows:

    $ java sushi.Main <options>
    
If you launch SUSHI without options it will print a help screen that lists all the available options. The indispensable ones, that you *must* set in order for SUSHI to work, are:

* `-classes`: a semicolon separated list of paths; It is the classpath of the software to test.
* `-target_class`: the name in [internal classfile format](http://docs.oracle.com/javase/specs/jvms/se6/html/ClassFile.doc.html#14757) of the class to test: SUSHI will generate tests for all the methods in the class. Or alternatively:
* `-target_method`: the signature of a method to test. The signature is a semicolon-separated list of: the name of the container class in internal classfile format; the [descriptor](http://docs.oracle.com/javase/specs/jvms/se6/html/ClassFile.doc.html#1169) of the method; the name of the method. You can use the `javap` command, included with every JDK setup, to obtain the internal format signatures of methods: `javap -s my.Class` prints the list of all the methods in `my.Class` with their signatures in internal format.
* `-evosuite`: the path of the EvoSuite jar file contained in the `evosuite/` folder.
* `-jbse_lib`: this must be set to the path of the `jbse.jar` file.
* `-sushi_lib`: this must be set to the path of the `sushi-lib.jar` file.
* `-z3`:  the path to the Z3 binary (you can omit it if Z3 is on the system PATH).
* `-tmp_base`: a path to a temporary directory; SUSHI needs to create many files for its intermediate results, and 
will put them in a subdirectory of `-tmp_base` having as name the date and time it was launched.
* `-out`: a path to a directory where the generated tests will be put.

An alternative way to configure SUSHI is to define a subclass of the class `sushi.configure.ParametersModifier` contained 
in the sushi-lib subproject. The subclass should override one or more of the `modify` methods that receive as input a  parameter object, and modify the object
by setting the parameters of interest. In this case SUSHI must be invoked by specifying the following options:

* `-params_modifier_path`: the path where your custom subclass of  `sushi.configure.ParametersModifier` is.
* `-params_modifier_class`: the name of your custom subclass of  `sushi.configure.ParametersModifier`.

You will find examples of this way of configuring SUSHI in the sushi-experiment, sushi-experiment-closure01 and sushi-experiment-closure72 projects.

## Generated tests
The tests are generated in EvoSuite format: Each suite produces two classes,
one scaffolding class and the suite containing all the test cases. SUSHI
will produce many suites each containing exactly one test, so if SUSHI
generates, e.g., 10 test cases then in the directory indicated with the 
`-out` command line parameter you will find 10 scaffolding classes and
10 test suites with 1 test case each. Note that you do *not* need the 
scaffolding to compile and run the test suites, but the test suites 
depend on junit and on the EvoSuite jar. You can safely remove the 
latter dependency by manually editing the generated files, otherwise
you need to put the EvoSuite jar included with this distribution in 
the classpath when compiling the generated test suites.

The generated files have names structured as follows:
    
    <class name>_<method name>_PC_<number>_<number>_Test_scaffolding.java //the scaffolding
    <class name>_<method name>_PC_<number>_<number>_Test.java             //the suite

where `<class name>` is the name of the class under test, `<method name>` is the name
of the method under test, and the `PC_<number>_<number>` identifies the trace along 
which the test executes (you don’t need this information so we will not elaborate
on it further). Note that the scaffolding and test suite classes are declared in the 
same package as the class under test, so they can access the package-level
members of the class under test. This means, for example, that a generated .java files 
for an `avl_tree.AvlTree` class under test, if you have specified the option 
`-out /your/out/dir`, could be something like `/your/out/dir/avl_tree/AvlTree_findMax_PC_2_1_Test.java`. 
If you want to compile and execute the test suites add the right
root to the classpath and qualify the class name of the test suite with the 
package name, e.g.:


    $ javac -cp junit.jar:evosuite-shaded-1.0.3.jar:avltree.jar
        /your/out/dir/avl_tree/AvlTree_findMax_PC_2_1_Test.java
    $ java -cp junit.jar:evosuite-shaded-1.0.3.jar:avltree.jar:/your/out/dir
        org.junit.runner.JUnitCore avl_tree.AvlTree_findMax_PC_2_1_Test

## Disclaimer

SUSHI is a research prototype. As such, it is more focused on functionality than on usability. We are committed to progressively improve the situation.
