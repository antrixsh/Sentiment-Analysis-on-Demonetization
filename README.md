# Sentiment-Analysis-on-Demonetization
Sentiment Analysis on Demonetization by Using Apache Pig. Let's find out the views of different people on the demonetization by analysing the tweets from twitter. In Dataset twitter tweets are gathered in CSV format. 

## DataSet Link
https://goo.gl/WkXksE

## Afinn File Link
https://goo.gl/8CUnV3

## Step 1 
Create a project directory in hdfs and copy your data file in the same directory by using pig command
Refer this repository to know the commands
https://goo.gl/9MEqRj

Now we will load the data into pig using PigStorage as follows:

$load_tweets = LOAD '/demonetization-tweets.csv' USING PigStorage(',');

Now after loading successfully, you can see the tweets loaded successfully into pig by using the dump command or illustrate command. dump command takes much time as it create mapreduce job as we are testing the same so we also can use illustrate command.

#### Here is the sample tweet
("1","RT @rssurjewala: Critical question: Was PayTM informed about #Demonetization edict by PM? It's clearly fishy and requires full disclosure &amp;�",FALSE,0,NA,"2016-11-23 18:40:30",FALSE,NA,"801495656976318464",NA,"<a href=""http://twitter.com/download/android"" rel=""nofollow"">Twitter for Android</a>","HASHTAGFARZIWAL",331,TRUE,FALSE)

#### Data Set Description

    id,Text (Tweets),favorited,favoriteCount,replyToSN,created,truncated,replyToSID,id,replyToUID,statusSource,screenName, 
    retweetCount,isRetweet,retweeted

#### Now from this columns, we will extract the id and the tweet_text as follows

$extract_details = FOREACH load_tweets GENERATE $0 as id,$1 as text;

Now if you dump the extracted columns, you will get the id and the tweet_text as follows:
("1","RT @rssurjewala: Critical question: Was PayTM informed about #Demonetization edict by PM? It's clearly fishy and requires full disclosure &amp;�")

Now we will divide the tweet_text into words to calculate the sentiment of the whole tweet.
$tokens = foreach extract_details generate id,text, FLATTEN(TOKENIZE(text)) As word;

For every word in the tweet_text, each word will be taken and created as a new rowwe can use the dump command to check the same. Here is the sample.
("1","RT @rssurjewala: Critical question: Was PayTM informed about #Demonetization edict by PM? It's clearly fishy and requires full disclosure &amp;�",RT)

In the above sample record, we can see that at the last RT word has been taken and created a new record for that. we can use the describe tokens command to check the schema of that relation and is as follows:
tokens: {id: bytearray,text: bytearray,word: chararray}

#### Now, we have to analyse the Sentiment for the tweet by using the words in the text. We will rate the word as per its meaning from +5 to -5 using the dictionary AFINN. The AFINN is a dictionary which consists of 2500 words which are rated from +5 to -5 depending on their meaning. You can download the dictionary from the above link mentioned below data set link.

We will load the dictionary into pig by using the below statement:
$dictionary = load '/AFINN.txt' using PigStorage('\t') AS(word:chararray,rating:int);
We can see the contents of the AFINN dictionary in the below screen shot.
![capture4](https://user-images.githubusercontent.com/26787806/51831583-732c9680-2318-11e9-80e8-eb3283a9354d.PNG)

#### Now, let’s perform a map side join by joining the tokens statement and the dictionary contents using this relation:
word_rating = join tokens by word left outer, dictionary by word using 'replicated';

We can see the schema of the statement after performing join operation by using the below command:
describe word_rating;
word_rating: {tokens::id: bytearray,tokens::text: bytearray,tokens::word: chararray,dictionary::word: 
chararray,dictionary::rating: int}

In the above statement describe word_rating, we can see that the word_rating has joined the tokens (consists of id, tweet text, word) statement and the dictionary(consists of word, rating).

#### Now we will extract the id,tweet text and word rating(from the dictionary) by using the below relation.
rating = foreach word_rating generate tokens::id as id,tokens::text as text, dictionary::rating as rate;

We can now see the schema of the relation rating by using the command describe rating.
rating: {id: bytearray,text: bytearray,rate: int}

In the above statement describe rating we can see that our relation now consists of id,tweet text and rate(for each word).
#### Now, we will group the rating of all the words in a tweet by using the below relation:
word_group = group rating by (id,text);

Here we have grouped by two constraints, id and tweet text.
#### Now, let’s perform the Average operation on the rating of the words per each tweet.

avg_rate = foreach word_group generate group, AVG(rating.rate) as tweet_rating;
#### Now we have calculated the Average rating of the tweet using the rating of each word.

From the above relation, we will get all the tweets i.e., both positive and negative.
Here, we can classify the positive tweets by taking the rating of the tweet which can be from 0-5. We can classify the negative tweets by taking the rating of the tweet from -5 to -1.

We have now successfully performed the Sentiment Analysis on Twitter data using Pig. We now have the tweets and its rating, so let’s perform an operation to filter out the positive tweets.
#### Now we will filter the positive tweets using the below statement:

positive_tweets = filter avg_rate by tweet_rating>=0;

Here are the sample tweets with positive ratings.
(("7989","RT @rssurjewala: Critical question: Was PayTM informed about #Demonetization edict by PM? It's clearly fishy and requires full disclosure &amp;�"),1.0)
(("7990","All weddings now need to be approved by RBI... Amazing times #demonetization isn't that what we are understanding"),2.0)
(("7993","RT @jackerhack: Indore's collector would like you to shut up about #demonetization. At @internetfreedom we think that is a problem. https:/�"),2.0)
(("7994","@quizderek Post #Dmonetization the result will be totally different.The win is not because of #demonetization an all knows about it"),4.0)
(("7995","@baliramsingh2 So many restrictions. Not easy to avail the facility by anyone. Multiple U-turns by GOI on the issue. #DeMonetization #RBI"),1.0)
((How long, successful and sustainable will be this strategic game of #DeMonetization against Demons?"),3.0)
((No there r many, we cal them by many names like C#%),2.0)
((Akhilesh=not good,black money is good),3.0)
((And respect their decision,but support oppositio�"),2.0)
((And respect their decision,but support opposition just b'coz of party"),2.0)
(( the avg indian wants corruptn free india.. So in d name of black money, everybody agrees),1.0)

#### Like this we will also filter the negative tweets as follows:
negative_tweets = filter avg_rate by tweet_rating<0;
Here are the sample tweets with negative rating

(("7969","OK � now don�t complain that modi ji promised 2 Crore jobs a year but did only 1.35 Lakh. He is making up for thru� https://t.co/RiON3cqAlH"),-0.5)
(("7997","RT @sukanyaiyer2: #DeMonetization AAP protests by marching Against Govts move over DeMonetization &amp; he is also detained as he Tried 2 March�"),-2.0)
(("7998","#demonetization will help combat terror because Pak won't be able to print new notes! And now),-0.6666666666666666)
(("8000","RT @UnSubtleDesi: Kejriwal posts pic of dead robber and claims it's #demonetization related death? How shameless has this man become? https�"),-2.5)
((Only noise, chaos &amp; disruptions by obstructionist #�"),-2.0)
((Only noise, chaos &amp; disruptions by obstructi� https://t.co/zVE7MYt04G"),-2.0)
((5% bad idea, poor implementation"),-2.0)
((25% good idea, poor implementation),-2.0)
((If not for Aam Aadmi, listen to them no PM Modi?"),-1.0)
((Aim of #demonetization laudable, but Govt has no road map2create... https://t.co/A4Geu9chOv"),-1.0)
((Enough jokes on #Demonetization, also no more posts on politics or social affairs...),-1.0)
((RT @kanimozhi: Everyone seems to hate the rich, even the rich hates richer and the richer hates the richest. #Demonetization"),-1.3333333333333333)

Like this, you can perform sentiment analysis using Pig.

I hope that this blog helped you in understanding how to perform sentiment analysis on the views of different people using Pig.
