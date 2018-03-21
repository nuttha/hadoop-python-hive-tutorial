## MapReduce with Python

#### Start the cluster!

    /usr/local/hadoop/sbin/start-dfs.sh

Yes!!! It’s running. You can check the report on the cluster at this
address on your web browser:

    http://<YOUR_CLOUD_SERVER_IP>:9870

(Replace <YOUR_CLOUD_SERVER_ID> with the actual ip, like 167.214.312.54)    
On the terminal,

    jps

will show you that `DataNode`, `NameNode` and `SecondaryNameNode` are running.

#### Let’s stop it

    /usr/local/hadoop/sbin/stop-dfs.sh

Now go check

    http://<YOUR_CLOUD_SERVER_IP>:9870

It shouldn’t be there anymore!

#### Get data

Alright, let’s put some data in.

Let’s make a directory for these

    mkdir -p /home/hduser/textdata

First we’ll start with putting the data into our normal data system.
If you have some text files, you can use them for this.
If not, here are three ebooks (plain text `utf-8` encoding) you can
`wget`:

Ulyses by James Joyce    
[http://www.gutenberg.org/cache/epub/4300/pg4300.txt][1]   

Notebooks of Leonardo Da Vinci    
[http://www.gutenberg.org/cache/epub/5000/pg5000.txt][2]   

The Outline of Science by J Arthur Thomson    
[http://www.gutenberg.org/cache/epub/20417/pg20417.txt][3]   

(For example, to get these, you can do this:

    cd /home/hduser/textdata
    wget http://www.gutenberg.org/cache/epub/4300/pg4300.txt
    wget http://www.gutenberg.org/cache/epub/5000/pg5000.txt
    wget http://www.gutenberg.org/cache/epub/20417/pg20417.txt
    
)

#### Put data in hdfs

First, let’s start the cluster again!

    /usr/local/hadoop/sbin/start-dfs.sh

make some directories **in the hadoop distributed file system!**

    hdfs dfs -mkdir /user/
    hdfs dfs -mkdir /user/nuttha/

Of course replace `nuttha` with your own username.
Let’s check that they exist

    hdfs dfs -ls /
    hdfs dfs -ls /user/

Yay!

Ok, put some data in

    hdfs dfs -put /home/hduser/textdata/* /user/nuttha

Check and make sure it is in the hdfs

    hdfs dfs -ls /user/nuttha

Yay!

####Our mapper and reducer

Our mapper `mapper.py` includes the following code:
```python
#!/usr/bin/env python
"""A more advanced Mapper, using Python iterators and generators."""

import sys

def read_input(file):
    for line in file:
        # split the line into words
        yield line.split()

def main(separator='\t'):
    # input comes from STDIN (standard input)
    data = read_input(sys.stdin)
    for words in data:
        # write the results to STDOUT (standard output);
        # what we output here will be the input for the
        # Reduce step, i.e. the input for reducer.py
        #
        # tab-delimited; the trivial word count is 1
        for word in words:
            print '%s%s%d' % (word, separator, 1)

if __name__ == "__main__":
    main()
```

And our reducer `reducer.py` looks like this:
```python
#!/usr/bin/env python
"""A more advanced Reducer, using Python iterators and generators."""

from itertools import groupby
from operator import itemgetter
import sys

def read_mapper_output(file, separator='\t'):
    for line in file:
        yield line.rstrip().split(separator, 1)

def main(separator='\t'):
    # input comes from STDIN (standard input)
    data = read_mapper_output(sys.stdin, separator=separator)
    # groupby groups multiple word-count pairs by word,
    # and creates an iterator that returns consecutive keys and their group:
    #   current_word - string containing a word (the key)
    #   group - iterator yielding all ["&lt;current_word&gt;", "&lt;count&gt;"] items
    for current_word, group in groupby(data, itemgetter(0)):
        try:
            total_count = sum(int(count) for current_word, count in group)
            print "%s%s%d" % (current_word, separator, total_count)
        except ValueError:
            # count was not a number, so silently discard this item
            pass

if __name__ == "__main__":
    main()
```

Before running these codes, we need to make sure that textblob has its nltk corpora downloaded, so that it can work without an error. To do that, execute this on the command line (as the hduser):

    python -m textblob.download_corpora

####Let's run it!

Before giving the following command, don't forget to replace the `/user/nuttha` path (in the hdfs) with your own version, and the paths to `mapper.py` and `reducer.py` (in your droplet's local filesystem) with your own versions.

    mapred streaming -mapper /home/hduser/mapper.py -reducer /home/hduser/reducer.py -input /user/nuttha/* -output /user/nuttha/book-output

Booom ! It's running.

#### Looking at the output
Once it's done,

    hdfs dfs -ls /user/nuttha/book-output

should show that there is a `_SUCCESS` file (showing we did it!) and
another file called `part-00000`

This `part-00000` is our output. To look in:

    hdfs dfs -cat /user/nuttha/book-output/part-00000

or just

    hdfs dfs -cat /user/nuttha/book-output/*

will show the output of our job!

If you want to see the most common words, run:

    hdfs dfs -cat /user/nuttha/book-output/* | sort -rnk2 | less

########Note:
If something went wrong when you ran your mapreduce job, you fix something and want to run it again, it will throw a different error, saying that the book-output directory already exists in hdfs. This error is thrown to avoid overwriting previous results. If you want to just rerun it anyway, you need to delete the output first, so it can be created again:

    hdfs dfs -rm -r /user/nuttha/book-output
    
    

[1]: http://www.gutenberg.org/cache/epub/4300/pg4300.txt
[2]: http://www.gutenberg.org/cache/epub/5000/pg5000.txt
[3]: http://www.gutenberg.org/cache/epub/20417/pg20417.txt
