public: yes
tags: [c++, gif, debug utils]
summary: |
  A small tutorial on how I added .gif recording ability into my game engine.
  
Recording a .gif from game
==========================

I recently wrote a small utility that captures a .gif animation directly from
the game. Here's a quick explonations of how I did it. Things to consider, I'm 
using C++, OpenGL and Windows. The following code snippets, libraries were used:

  - `Sean Barrett's stbiw <http://nothings.org/stb/stb_image_write.h>`_
  - `Image Magick <http://www.imagemagick.org/script/index.php>`_
  - `ExecuteProcess <http://goffconcepts.com/techarticles/development/cpp/createprocess.html>`_


The general principle is actually pretty simple. I just save every other screenshot into 
a folder, with a running number. After that, run the Image Magick script that 
produces a gif from the saved screenshots. 

Here's a bit of the details:


Capturing a screenshot with OpenGL and saving it with stbiw
-----------------------------------------------------------

`glReadPixels(...) <http://www.opengl.org/sdk/docs/man/xhtml/glReadPixels.xml>`_ is 
used to read the frame buffer. Unfortunately I noticed that glReadPixels doesn't
really work with all the parameters. If you want to save a smaller area of the 
screen there's a very high change of it just crashing. I read somewhere that
multiplies of 4 are only allowed, so I just decided to use glReadPixels to read
the whole frame buffer and manually crop the image to desired size. Not the most
efficient way of doing things, but it works. Here's my code for a taking a 
screenshot and saving it to a .png with `Sean Barrett's stbiw 
<http://nothings.org/stb/stb_image_write.h>`_

The full code can be found on github in the `poro framework <https://github.com/gummikana/poro/>`_ under 
`graphics_opengl.h <https://github.com/gummikana/poro/blob/master/source/poro/desktop/graphics_opengl.h>`_ and
`graphics_opengl.cpp <https://github.com/gummikana/poro/blob/master/source/poro/desktop/graphics_opengl.cpp>`_ 


.. code-block:: c

   
	void GraphicsOpenGL::SaveScreenshot( const std::string& filename, int pos_x, int pos_y, int w, int h )
	{
		int width = (int)mViewportSize.x;
		int height = (int)mViewportSize.y;

		static unsigned char* pixels = new unsigned char[ 3 * (int)mViewportSize.x * (int)mViewportSize.y ];

		// read the whole image into a buffer, since this crashes with unspecified sizes
		glReadPixels( (int)mViewportOffset.x, (int)mViewportOffset.y, (int)mViewportSize.x, (int)mViewportSize.y, GL_RGB, GL_UNSIGNED_BYTE, pixels);

		// need to flip the pixels
		for( int x = 0; x < width * 3; ++x ) 
		{
			for( int y = 0; y < height / 2; ++y ) 
			{
				std::swap( 
					pixels[ x + 3 * width * y ], 
					pixels[ x + 3 * width * ( height - y - 1 ) ] );
			}
		}
		

		unsigned char* other_pixels = pixels;

		// copy to other buffer
		if( pos_x != 0 || pos_y != 0 || w != width || height != h ) 
		{
			other_pixels = new unsigned char[ 3 * w * h ];	
			for( int y = 0; y < h; ++y )
			{
				for( int x = 0; x < w * 3; ++x )
				{
					int p1 = (x + 3 * w * y);
					int p2 = (x + (pos_x * 3)) + ( 3 * width * ( y + pos_y));
					other_pixels[ p1 ] = pixels[ p2 ];
				}
			}
		}

		int result = stbi_write_png( filename.c_str(), w, h, 3, other_pixels, w * 3 );
		if( result == 0 ) poro_logger << "Error SaveScreenshot() - couldn't write to file: " << filename << std::endl;

		if( other_pixels != pixels ) delete [] other_pixels;
		// no need to release the static pixels
		// delete [] pixels;	
	}
  


Saving every other frame
------------------------

The actual code for my Screenshotter class has some extra functionality that's 
probably not useful to anyone but me. Functionality like automatically saving
screenshots at certain times (one at the very beginning of running the program
and one at few minutes in) and copying those screenshots into a dropbox folder.
I might explain why that is at a later point. But the point is the code is
has a lot of extra functionality so if you're interested in seeing the full code
it's at github in here:

Here's the important parts:

.. code-block:: c

	void Screenshotter::OnKeyDown( int key, poro::types::charset unicode )
	{
		if( key == SDLK_F4 ) 
		{
			mDoingGifRecording = !mDoingGifRecording;
			
			// create the folder for the png files
			if( mDoingGifRecording == true ) 
			{
				time_t now = time(0);
				tm *ltm = localtime(&now);

				std::stringstream ss;
				ss << "screenshots_animated/anim_" << 1900 + ltm->tm_year 
					<< std::setfill( '0' ) << std::setw( 2 ) << 1 + ltm->tm_mon
					<< std::setfill( '0' ) << std::setw( 2 ) << ltm->tm_mday 
					<< "-" 
					<< std::setfill( '0' ) << std::setw( 2 ) << ltm->tm_hour
					<< std::setfill( '0' ) << std::setw( 2 ) << ltm->tm_min
					<< std::setfill( '0' ) << std::setw( 2 ) << ltm->tm_sec
					<< "-" << mFrameCount;

				// on Windows you can just run CreateDirectory( ss.str().c_str(), 0 );
				ceng::CreateDir( ss.str() );

				mGifFilePath = ss.str() + "/frame_";
			}
			else // the end of a gif recording
			{
				std::stringstream ss;
				ss << " -delay 1x30 " << mGifFilePath << "*.png " << ceng::GetParentPath( mGifFilePath ) << ".gif";
				ExecuteProcess( PATH_TO_IMAGEMAGICK, ss.str() );
			}
		}
	}


	void Screenshotter::Update( float dt )
	{
		mFrameCount++;

		// save every other frame
		if( mDoingGifRecording && mFrameCount % 2 == 0 ) 
		{
			types::irect temp_rect = GetIRect( mGifRectStartPos, mGifRectEndPos );
			DoScreenshot( mGifFilePath, false, &temp_rect );
		}
	}


	std::string Screenshotter::DoScreenshot( const std::string& prefix, bool add_path_before, const types::irect* rect  )
	{
		std::stringstream ss;
		ss << prefix << mFrameCount << ".png";

		std::string result = ss.str();
		Poro()->GetGraphics()->SaveScreenshot( result, rect->x, rect->y, rect->w, rect->h );
		return result;
	}


Running Image Magick Script
---------------------------

I'm using the `Windows binaries of Image Magick <http://www.imagemagick.org/script/binary-releases.php#windows>`_ 
to create the .gif animation from the .png files. The command line arguments are:

::

	convert -delay 1x30 anim_20130118-110021/*.png anim_20130118-110021.gif

The -delay 1x30  causes the animation to run at 30 fps. Since I'm running my game at 60 fps and recording only 
every other frame, this should produce nice and smooth .gif animations. 


To run the Image Magick script from C++, I use this 
`non blocking process creation function <http://goffconcepts.com/techarticles/development/cpp/createprocess.html>`_ 
(this works only on Windows). 

.. code-block:: c

	std::wstring s2ws(const std::string& s)
	{
		int len;
		int slength = (int)s.length() + 1;
		len = MultiByteToWideChar(CP_ACP, 0, s.c_str(), slength, 0, 0); 
		wchar_t* buf = new wchar_t[len];
		MultiByteToWideChar(CP_ACP, 0, s.c_str(), slength, buf, len);
		std::wstring r(buf);
		delete[] buf;
		return r;
	}

	// this is taken from here http://goffconcepts.com/techarticles/development/cpp/createprocess.html
	size_t ExecuteProcess( const std::string& full_path_to_exe, const std::string& params, size_t SecondsToWait = 500 ) 
	{ 

		std::wstring FullPathToExe = s2ws( full_path_to_exe );
		std::wstring Parameters = s2ws( params );

		size_t iMyCounter = 0, iReturnVal = 0, iPos = 0; 
		DWORD dwExitCode = 0; 
		std::wstring sTempStr = L""; 

		/* - NOTE - You should check here to see if the exe even exists */ 

		/* Add a space to the beginning of the Parameters */ 
		if (Parameters.size() != 0) 
		{ 
			if (Parameters[0] != L' ') 
			{ 
				Parameters.insert(0,L" "); 
			} 
		} 

		/* The first parameter needs to be the exe itself */ 
		sTempStr = FullPathToExe; 
		iPos = sTempStr.find_last_of(L"\\"); 
		sTempStr.erase(0, iPos +1); 
		Parameters = sTempStr.append(Parameters); 

		 /* CreateProcessW can modify Parameters thus we allocate needed memory */ 
		static wchar_t pwszParam[ 1024 ];
		// wchar_t * pwszParam = new wchar_t[Parameters.size() + 1]; 
		if (Parameters.size() > 1024 ) 
		{ 
			return 1; 
		} 
		const wchar_t* pchrTemp = Parameters.c_str(); 
		wcscpy_s(pwszParam, Parameters.size() + 1, pchrTemp); 

		/* CreateProcess API initialization */ 
		STARTUPINFOW siStartupInfo; 
		PROCESS_INFORMATION piProcessInfo; 
		memset(&siStartupInfo, 0, sizeof(siStartupInfo)); 
		memset(&piProcessInfo, 0, sizeof(piProcessInfo)); 
		siStartupInfo.cb = sizeof(siStartupInfo); 
		siStartupInfo.wShowWindow = 0;
		siStartupInfo.dwFlags = STARTF_FORCEOFFFEEDBACK;

		if(!CreateProcessW(const_cast<LPCWSTR>(FullPathToExe.c_str()), 
								pwszParam, 0, 0, false, 
								CREATE_DEFAULT_ERROR_MODE, 0, 0, 
								&siStartupInfo, &piProcessInfo ) ) 
		{ 
			 /* Watch the process. */ 
			/*
			dwExitCode = WaitForSingleObject(piProcessInfo.hProcess, (SecondsToWait * 1000)); 
		} 
		else 
		{ */
			/* CreateProcess failed */ 
			iReturnVal = GetLastError(); 
		} 

		/* Free memory */ 
		// delete[]pwszParam; 
		// pwszParam = 0; 

		/* Release handles */ 
		// CloseHandle(piProcessInfo.hProcess); 
		// CloseHandle(piProcessInfo.hThread); 

		return iReturnVal; 
	} 



Some extra functionality
------------------------

I also found that parsing a gif animation with full screen resolution isn't the 
brightest idea. First of all Image Magick crashes if it runs out of memory and
.gif animation produced with this technique would take too much bandwidth to be
useful. 

So my solution to this was to allow the user to specifiy an area of the screen 
that's captured. This is done by holding down the F3 key and drawing an area
with the mouse. I'm pretty sure any capable programmer can write this 
functionality in 15 minutes. The other solution to this problem would be to
scale down the images saved. This could also be done with Image Magick or by 
code when the screenshots are being saved. I leave these problems to you to 
solve :) For my current project I only need to capture small areas of the 
screen, so resizing wouldn't be of much use.


