public: yes
tags: [todo.txt, devlog, utilities]
summary: |
  TODO.txt parsing for productivity logging
  
Automatic Dev Log from TODO.txt (and a game)
============================================

I created a PHP script that parses my TODO.txt and automagically creates daily dev logs, 
complete with screenshots (taken automatically) from the game. The full 
source code of `devlogger can be found on github <https://github.com/gummikana/devlogger>`_ .

The problem
-----------

Dev logs or development logs have become somewhat popular amongst game developers. 
Especially now with the rise of alpha games and open game development it's ever 
more important to communicate to your players that progress is being made on the 
game. Even if no one has updated the blog in a year.

The problem with maintaining a dev log or even changes.txt is that it is extra work 
(it doesn't really get the game closer to being done). The friction of writing a 
separate list is usually enough to make me drift away from doing it.

To solve this problem I created few scripts that automate the process.

Genesis
-------

The genesis of these automatic dev logs is really in post-it notes.
Long ago I really enjoyed using `post-it <http://www.kloonigames.com/blog/wp-content/uploads/2009/01/dev_img_03.jpg>`_ 
`notes <http://www.kloonigames.com/blog/wp-content/uploads/2009/01/dev_img_01.jpg>`_ to track tasks 
that had to be done. This analog method had couple of advantages to it. First it worked as an easy way 
to write down tasks and mark them done and have them be easily glanced. Also a big part of it was the sense of 
progress you'd get as the post-its would reproduce around your monitor.

Unfortunately the world moved on from bulky and fast response CRT monitors into the new age of slow response flat screens with 
very tiny none existing borders. The post-it method had a fall down. Quite literally. 

Instead I started writing my TODO's into a simple text file with notepad. This worked very well for me. I tried couple of 
"real" applications, but they all felt horrible and usually ran in the browser. The browser is the worst place to
put a TODO application, since it's such a tiny step (CTRL + T) to go from selecting next task to work on, to a hour long
bindge of looking for inspiration in the form of cat pictures. 

The plain text TODO.txt served me very well. Especially after I started sharing and keeping it in synch with 
`Dropbox <https://www.dropbox.com/>`_ .

The problem with the TODO.txt was that it lacked the sense of progress that you'd get from the post-it notes. As the file would
grow in size, it would get too big to be useful. After editing it down to just a handful of unfinished tasks it felt like all
my progress on the project was erased. 

At the same time I was wondering if I could do some kind of productivity tracker tool. I've experimented with these before, 
but they have the same problem as dev logging. If I have to do anything extra to use them, they fall by the way side. Then 
it hit me: I could parse the TODO.txt with an external program and use it to track my progress. Just to get some kind of 
stats about my working habbits. Dropbox proved to be very useful for this, since I could share my TODO.txt with the internet and
I could just write a small script that would run on a server and read the file from time to time and parse it.

From there the idea of automatically creating dev logs was a very tiny step.


The other part of this whole process lied in another productivity and process tracking method I had written. The idea was simply
to save random screenshot from the game everytime I ran it. This way I could get a visual look into how things very progressing. 
This also fit nicely into the game I was making, since the visual difference from random to random shot was quite big. This proved
to be a very fun and useful way to get somekind of sense of progress. Then I figured I could just pick one of these random shots
every day and I'd have a fully automated dev logs. And that's what I did.

Another unespected perk of this system was, that I know have a complete history of all my TODO.txts. There always seems to be 
horrible feeling of loss when you destroy or remove completed tasks from TODO.txts. I've even know to carry some of the post-it
notes from previous projects into new appartements that I've moved into. This knowledge that I have the complete history of
all the TODO.txts makes it much easier to edit and remove things from my TODO.txts.

Technical Details
-----------------

I have a web server that runs few PHP scripts and a MYSQL database. Into this database all the tasks are entered and parsed.
On the server I keep a daily copy of the actual TODO.txt. 





Code: Serializing basics
------------------------

.. code-block:: c

	struct RandomClass {
		int number_param;
		std::string text_param;
		std::string lots_of_text;

		void Serialize( ceng::CXmlFileSys* filesys ) {	
			XML_BindAttribute( filesys, number_param );
			XML_BindAttributeAlias( filesys, text_param, "text" );
			XML_Bind( filesys, lots_of_text );
		}
	};


That's it. That's the way you serialize the data. Note that you don't have to inherit from a base class. You only need to have a public function void Serialize( ceng::CXmlFileSys* filesys ) and you're good to go. There are two ways you can serialize data. You can "bind" them as attributes using XML_BindAttribute or XML_BindAttributeAlias -macros. The Alias version of the macro lets you define a separate name for the attribute. If you use XML_BindAttribute, the name of variable will be used. XML_Bind will write a new child node with the name of the variable. XML_BindAlias will let you give a name to the child node. You can use XML_Bind also to serialize other classes. 


Code: Saving and loading
------------------------

Here's how you load and save an object into a file with ceng_xml.


.. code-block:: c

	RandomClass rc;
	ceng::XmlLoadFromFile( rc, "random_class.xml", "RandomClass" ); 
	ceng::XmlSaveToFile( rc, "random_class.xml", "RandomClass" ); 


The output of this looks like this. Note not initializing the data causes some strange values to appear.

.. code-block:: xml

	<RandomClass number_param="12345" text="this is text" >
	  <lots_of_text>
	    This is also text
	  </lots_of_text>
	</RandomClass>


Code: Serializing advanced
--------------------------

.. code-block:: c

	struct AnotherClass {
		bool		var_bool;
		float		var_float;
		double		var_double;

		void Serialize( ceng::CXmlFileSys* filesys ) {
			XML_BindAttribute( filesys, var_bool );
			XML_BindAttribute( filesys, var_float );
			XML_BindAttribute( filesys, var_double );
		}
	};


	class CompositeClass {
	public:
		CompositeClass() : child_2( NULL ) { }
		
		AnotherClass	child_1;
		AnotherClass*	child_2;
		
		void Serialize( ceng::CXmlFileSys* filesys ) {	
			XML_BindAlias( filesys, child_1, "Child1" );
			XML_BindPtrAlias( filesys, child_2, "Child2" );		// note the different function used
		}
	};

The XML_BindPtrAlias is function that creates or destroyes the pointer if the node is found in the xml file or not.


Details
-------

There really two parts to this whole operation. First order of business is parsing open an xml file and creating the CXmlNode tree structure that corresponds to the xml file. The second part is serializing the data into or from the CXmlNode tree structure.

Parsing the XML file
--------------------

Parsing an xml file is done with the CXmlParser class. It calls the handler it's given with functions like: StartElement(...), EndElement(...) and it's the job of the handler to create an CXmlNode tree structure from the data the parser passes to it. 

.. code-block:: c

	CXmlNode* ParseXmlFile( string file ) {
		CXmlParser parser;
		CXmlHandler handler;

		parser.SetHandler( &handler );
		parser.ParseFile( file.c_str() );

		CXmlNode* root_node = handler.GetRootElement();
		return root_node;
	}


Side note: This interface should allow for other file types to be used instead of xml. Just write a new parser for the file type and it should work. Even existing parser could be plugged into this, like the industry's standard XML parser TinyXML (or TinyXML2). Or we could extend this to use JSON since that seems to be hot right now. Also a binary format could be nice as well... If someone is brave enough to give these a try, let me know :)

Saving to an XML file
---------------------

Saving is actually done a bit differently. Since there's really no need parse anything, saving is just done with the CXmlStreamHandler class. 

.. code-block:: c

	void SaveToXml( CXmlNode* node, string file ) {
		ofstream file_output( file.c_str(), ios::out );

		CXmlStreamHandler handler;
		handler.ParseOpen( node, file_output );

		file_output.close();
	}
	

Here's the CXmlStreamHandler::ParseOpen(...) - function which recursivly calls it's self and parses the tree.

.. code-block:: c

	void ParseOpen( CXmlNode* rootnode, std::ostream& stream )
	{
		StartElement( rootnode->GetName(), CreateAttributes( rootnode ), stream );
		Characters( rootnode->GetContent() , stream );
		for( int i = 0; i < rootnode->GetChildCount(); i++ )
			ParseOpen( rootnode->GetChild( i ), stream );

		EndElement( rootnode->GetName(), stream );
	}
	
	
----

The problems
* XML Format, XML has its benefits. It's human readable, easily editable and looks nice. The problems with it are pretty horrible as well. There's a lot of duplication of data. A lot. Especially for larger amounts of data, the amount of disk space required can easily be 3x as much. But the biggest problem with it is that it's really slow to parse. I'm using a custom parser that I've written and it's been the biggest source of pain in using ceng_xml. The amount of bugs that crash the system or cause an infinite loop have been quite the source of pain. The speed of parsing that rarely been an issue, but when you move away from the PC world it can easily become one. As was the case of porting Crayon Physics Deluxe to the iPad. There was quite a bit of rewriting that happened. 

The other problem with ceng_xml is that it's quite liberous with it's memory use. It creates quite a bit small objects and that can easily cause memory fragmentation. This has been an issue couple of times and I've tryid to circumvent that by using a memory pool. That did the trick, but I'm not too happy with that part either. 

If there is someone who wants to integrate tiny_xml into ceng_xml, that could be very useful. Also other file formats could potentially be supported, but I haven't really put in the time to do that. 



Why XML? 
Well to be completely honest I'm not too happy with XML format. It creates a lot of duplication of data, and parsing it takes quite a lot of CPU cycles. 