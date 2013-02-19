public: yes
tags: [c++, xml, debug utils, serialization, utilities]
summary: |
  ceng_xml - a C++ tool to serialize data into and out from an xml file.
  
ceng_xml - C++ tool to serialize data into an xml  
=================================================

There are certain utilities that a programmer uses quite often. One such thing for me is a collection of C++ classes that aid in the serialization of data to an xml file and from there. I shared these tools quite a long time ago, but I never made any kind of noise of them, so I guess no one else uses them. They've served me well so I thought I'd give a little back to the open source community that has given me so much.

code example #1
---------------

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


That's it. That's the way you serialize the data.


code example #2
---------------

Here's how you save and load an object into a file with ceng_xml.


.. code-block:: c

	RandomClass rc;
	ceng::XmlSaveToFile( rc, "random_class.xml", "RandomClass" ); 
	ceng::XmlLoadFromFile( rc, "random_class.xml", "RandomClass" ); 


The output of this looks like this. Note not initializing the data causes some strange values to appear.

.. code-block:: xml

	<RandomClass number_param="61214699" text="" >
	  <lots_of_text>
	  </lots_of_text>
	</RandomClass>



Here's the git repository of ceng_xml files...

The ease of loading data with this utility has served me well. I think I've used it in every single game that I've created in C++. I'll go into a bit more detail in here. 

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