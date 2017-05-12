## Tutorial 101: PageRank Example in Spark 2.0
### Understanding the Algorithm & Spark Code Implementation
 
  The Apache Spark PageRank is a good example for learning how to use Spark. The sample program computes the PageRank of URLs from an input file which should be in format of:
   url_1   url_4
   url_2   url_1
   url_3   url_2
   url_3   url_1
   url_4   url_3
   url_4   url_1  

where each URL and their neighbors are separated by space(s). The above input data can be represented by the following graph. For example, URL_3 references URL_1 & URL_2 while it is referenced by URL_4.  

<img src="/images/img-1.jpg" width="447" height="383">

The SparkPageRank.scala code looks deceivingly simple but to understand how things actually work requires a deeper understanding of Spark RDDs and its Scala based functional API. In a previous article I have described the steps required to setup the project in Scala IDE for Eclipse and run the code on Hortonworks 2.5 Sandbox. Here are shall take a deep dive into how the algorithm works to uncover its implementation detail and how it actually runs within SPARK. 

### How the Algorithm Works
The PageRank algorithm outputs a probability distribution used to represent the likelihood that a person randomly clicking on web page links will arrive at a particular web page. If we run the PageRank program with the input data file and indicate 20 iterations we shall get the following output:

url_4 has rank: 1.3705281840649928. <\br>
url_2 has rank: 0.4613200524321036. <\br>
url_3 has rank: 0.7323900229505396. <\br>
url_1 has rank: 1.4357617405523626. <\br>

The results clearly indicates that URL_1 has the highest page rank followed by URL_4 and then URL_3 & last URL_2. The algorithm works in the following manner:

If a URL (page) is referenced by other URLs then its rank increases because being referenced means you are important which is the case of URL_1. While if an important URL like URL_1 references other URLs this will increase the destination’s ranking which is the case of URL_4 that is referenced by URL_1; that is the reason why URL_4 ranking is higher than the other two URLs (URL_2 & URL_3). If we look at the various arrows in the above diagram we can see that URL_2 is referenced the least and that is the reason why it has the lowest ranking.

The rest of the article will take a deeper look at the Scala code that implements the algorithm. The code is made of 3 main parts. 

SparkPageRank Program: Part#1
To run the PageRank program you need to pass the class name, jar location, input data file and number of iterations. The command looks like the following (please refer to the Project Setup article): 

./bin/spark-submit --class com.scalaproj.SparkPageRank --master yarn --num-executors 1 --driver-memory 512m --executor-memory 512m --executor-cores 1 ~/testing/jars/pagerank.jar /input/pagerank_data.txt 20 

PageRank program is made of two parts. The first part is as follows:

(1)    val iters = if (args.length > 1) args(1).toInt else 10   // sets iteration from argument

(2)    val lines = spark.read.textFile(args(0)).rdd    // reading text file into Dataset[String] -> RDD1

(3)    val pairs = lines.map{ s =>

           val parts = s.split("\\s+")

(4)              (parts(0), parts(1))                 // create the tuple <url, url> for each line in the file.

           }

(5)    val links = pairs.distinct().groupByKey().cache()   // RDD1 <string, string> -> RDD2<string, iterable>   

The first line of the code above sets the iteration from the argument. The second line reads the input data file (file name passed as the arg0)) produces a Dataset of strings and then transforms the Dataset into an RDD where each line in the file is one whole string within the RDD. You can think of an RDD is a sort of list that is special to Spark because the data within the RDD is distributed among the various nodes. 

Please note that I have introduced a pair variable into the original code to make the program more readable.

Line #3 & 4 in the above code splits the string (or whole line in the input file) into a tuple of pairs of two URL strings. Once the whole file is split an array is generated with two elements for each line. In Line#4 of the code is where the transformation occurs from the array to the tuple.

and the first RDD of pairs are created the program runs a groupByKey command to produce a second RDD (links). The resultant links for the input data is as follows:

 Key     Array (iterable)
 url_4   [url_3, url_1]
 url_3   [url_2, url_1]
 url_2   [url_1]
 url_1   [url_4]
 
Actually the Array in the above table is not a true array it is an iterator on the resultant array of urls. This is what the groupByKey command produces when applied on an RDD. This is an important and powerful construct in Spark and every programmer needs to understand it well.

### SparkPageRank Program: Part#2
 

The code in this part is made of a single line:

  var ranks = links.mapValues(v => 1.0)    // create a new RDD <key,one>

which creates a third RDD that is also made of a tuples of pairs, but in this case a string & double pair. 

  Key    Value (Double) 
  url_4   1.0
  url_3   1.0
  url_2   1.0
  url_1   1.0
 

The ranks RDD is initially populated by 1.0 for all the URLs. In the next part of the code sample we shall see how this ranks RDD will change to become eventually the end result page rank we mentioned above.  

### SparkPageRank Program: Part#3
 

Here is the core of the algorithm and where 

