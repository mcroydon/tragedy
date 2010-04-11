# Tragedy

Tragedy is a high-level Cassandra Object Abstraction for Python.

## Tragedy's Data Model

In Tragedy you build your data model from Models and Indexes. An abstract *Model* specifies the kind data that can be stored in a Model-Instance. We also call a Model-Instance a Row, since specific Model-Instances are uniquely identified by their unique RowKey. Knowing the Model and RowKey is all you need to store and retrieve data from Cassandra. The attributes of the Model correspond to the Columns of a Row. Each Column has a Field-Type like StringField or IntegerField. The RowKey decides which specific Row/Model-Instance the user is referring to and on which physical machine the data is stored. If you lose a RowKey, you can never store or retrieve that data again. Any Unicode string can be used as RowKey as long as it is unique among all Rows of a Model. If there's no naturally unique identifier for the data in a Row, you can ask Tragedy to generate a UUID-RowKey for you.

An *Index* is a special kind of Model with an unlimited number of Columns that all have the same Field-Type (usually ForeignKey). Indexes are used to map from one RowKey (e.g. an Username), to an ordered list of many others (e.g. a list of Blogposts). The Index is accessed with a RowKey, and doesn't store any data except for the ordered list of RowKeys to other Models.

Since distributed datastores like Cassandra don't support queries other than retrieving Models by RowKey, you have to create your Indexes when you write your data. By carefully tying Models and Indexes together, you can build complex but efficient applications that can run on large computing clusters.

Here's a simple example. Let's define and store a Tweet for a twitter-like application:
	class Tweet(Model):
    	uuid    = RowKey(autogenerate=True) # generate a UUID for us.
    	message = StringField()    
    	author  = ForeignKey(foreign_class=User, mandatory=True)

Tweet is a Model specification. If we instantiate Tweet, we get a specific tweet that we can write to the database:

    new_tweet = Tweet(message="Twittering from tragedy!", author='merlin')
	new_tweet.save()

Tweet instances are referred to and accessed by a RowKey. Tweet's RowKey is named `uuid` and its value is (randomly) autogenerated on save. Object can only be retrieved from the datastore if their RowKey is known. Since Tweet's RowKey is random, we'll lose the Tweet if we don't keep a reference somehow. One way to do this, is to store the RowKey in an Index. Let's create an Index of all tweets a specific user posts:

	class TweetsSent(Index):
    	by_username = RowKey()
    	targetmodel = ForeignKey(foreign_class=Tweet, compare_with='TimeUUIDType')

	merlinIndex = TweetsSent(by_username='merlin')
	merlinIndex.append(new_tweet)
	merlinIndex.save()

TweetsSent is an abstract Index over Tweets sorted by Cassandra's TimeUUIDType. merlinIndex is a specifc TweetsSent-Index for user 'merlin', as specified by the given RowKey during instantiation. Items can be added to an Index using the .append() method, and changes to them saved using the .save() method. Just as with models, we can only retrieve Indexes whose RowKey we know. If we do, we can use .load() to load the index from the Database:

    tweets_by_user = TweetsSent(by_username='merlin').load()
	print tweets_by_user

The main difference between Indexes and Models is that Indexes keep track of an unlimited amount of ordered data of the same kind (normally ForeignKeys), whereas a Model keeps track of a limited number of data that can be any mixture of types. Indexes are most often used to to help us find Data whose RowKey we've forgotten. Models can refer to Indexes using ForeignKeys, and Indexes can refer to both Models and (less often) other Indexes. The call above gives us a list of Tweets previously posted by user 'merlin' with their RowKeys correctly set. However, since the Index only contains references the actual tweet data hasn't been loaded yet at this point. If we tried to work with those tweets, we'd see #MISSING# fields all over the place:

    [<Tweet 8649a1ca4ab843b9afa6cc954908ac04: {'message': '#MISSING#', 'author': '#MISSING#'}, ...]

To actually load the tweets we need to resolve them (retrieve them using their RowKeys). Luckily Indexes have the .resolve() helper to make this easy:

	tweets_by_user.resolve()
	print tweets_by_user
	[<Tweet ced314748d574379a817e1a1c9149789: {'message': "some message", 'author': <User merlin: {'password': '#MISSING#'}>}>

Behind the scenes Index.resolve() almost works like calling Model.load() on all Tweets in the list. It's more efficient though, since this combines all required queries into one multiquery for faster processing. Now we've seen how to create tweets, store them, and find them again. If you want to see how you can distribute them to Followers, scroll down for a full example of a twitter-like application.

That's about it for the basics. There's more stuff Tragedy can do for you, like automatic validation that Tragedy and Cassandra agree on the Data Model, and the following example shows of some of them. And get in touch if you have questions!

## Installation
  $ setup.py install   (optionally --cassandra to install the compiled cassandra thrift bindings)

## IRC and Contact
Come hang out on #cassandra on irc.freenode.net. My email address is enki@bbq.io.

## Example (full twitter-demo)

    import tragedy
    client = tragedy.connect(['localhost:9160'])
    from tragedy import *
    
    twitty_cluster  = Cluster('Twitty Cluster')
    twitty_keyspace = Keyspace('Twitty', twitty_cluster)
    
    ALLTWEETS_KEY = '!ALLTWEETS!' # virtual user that receives all tweets
    
    class User(Model):
        """A Model is stored and retrieved by its RowKey.
           Every Model has exactly one RowKey and one or more other Fields"""
        username  = RowKey()
        firstname = StringField(mandatory=False)
        lastname  = StringField(mandatory=False) # normally fields are mandatory
        password  = StringField()
    
        def follow(self, *one_or_more):
            fol = Following(username=self)
            for other in one_or_more:
                fol.append(other)
                FollowedBy(username=other).append(self).save()
            fol.save()
    
        def tweet(self, message):
            new_tweet = Tweet(author=self, message=message[:140]).save()
            TweetsSent(by_username=self).append(new_tweet).save()
            
            tr = TweetsReceived(by_username=ALLTWEETS_KEY)
            tr.append(new_tweet).save()
            
            for follower in self.get_followed_by():
                follower.receive(new_tweet)            
    
        def receive(self, tweet):
            TweetsReceived(by_username=self).append(tweet).save()
    
        def get_followed_by(self, *args, **kwargs):
            return FollowedBy(username=self).load(*args, **kwargs)
    
        def get_following(self, *args, **kwargs):
            return Following(username=self).load(*args, **kwargs)
    
        def get_tweets_sent(self, *args, **kwargs):
            return TweetsSent(by_username=self).load(*args, **kwargs).resolve()
    
        def get_tweets_received(self, *args, **kwargs):
            return TweetsSent(by_username=self).load(*args, **kwargs).resolve()
    
    class Tweet(Model):
        uuid    = RowKey(autogenerate=True) # generate a UUID for us.
        message = StringField()    
        author  = ForeignKey(foreign_class=User, mandatory=True)
    
        @staticmethod
        def get_recent_tweets(*args, **kwargs):
            tr = TweetsReceived(by_username=ALLTWEETS_KEY)
            return tr.load(*args, **kwargs).loadIterValues()
    
        # def __repr__(self):
        #     return '<%s> %s' % (self['author']['username'], self['message'])
    
    class TweetsSent(Index):
        """An index is an ordered mapping from a RowKey to
           instances of a specific Model."""
        by_username = RowKey()
        targetmodel = ForeignKey(foreign_class=Tweet, compare_with='TimeUUIDType')
    
    class TweetsReceived(Index):
        by_username = RowKey()
        targetmodel = ForeignKey(foreign_class=Tweet, compare_with='TimeUUIDType')
    
    class Following(Index):
        username = RowKey()
        targetmodel = ForeignKey(foreign_class=User, compare_with='TimeUUIDType', 
                                 unique=True)    
    
    class FollowedBy(Index):
        username = RowKey()
        targetmodel = ForeignKey(foreign_class=User, compare_with='TimeUUIDType',
                                 unique=True)
    
    # We're done with defining the Data Model. Let's verify that Cassandra agrees on the model!
    twitty_keyspace.verify_datamodel()
    # Ok, all set. Let's go!
    
    dave = User(username='dave', firstname='dave', password='test').save()
    merlin = User(username='merlin', firstname='merlin', password='sunshine').save()
    peter = User(username='peter', firstname='Peter', password='secret').save()
    
    dave.follow(merlin, peter)
    peter.follow(merlin)
    merlin.follow(dave)
    
    merlin.tweet("i've just started using twitty. send me a message!")
    dave.tweet('making breakfast')
    peter.tweet('sitting at home being bored')
    
    for dude in (dave,peter,merlin):
        name = dude['username']
        print '%s has these followers:' % (name,), dude.get_followed_by().values()
        print '%s follows' % (name,), dude.get_following().values()
        print '%s sent' % (name,), [x for x in dude.get_tweets_sent(count=3)]
        print '%s received' % (name,), [x for x in dude.get_tweets_received(count=3)]