## OpenGL with windows

Using CWnd for the window functions
Using OpenGL for rendering the contents in the window

### set up openGL for the window
	1. get the window hDC
	2. choose the pixel format
	3. create render context hRC
```cpp
	hDC=::GetDC(m_hWnd);

	PIXELFORMATDESCRIPTOR pixelDesc =
	{
		sizeof(PIXELFORMATDESCRIPTOR),
			1,
			PFD_DRAW_TO_WINDOW|PFD_SUPPORT_OPENGL|
			PFD_DOUBLEBUFFER,
			PFD_TYPE_RGBA,
			24,
			0,0,0,0,0,0,
			0,
			0,
			0,
			0,0,0,0,
			32,//
			0,
			0,
			PFD_MAIN_PLANE,
			0,
			0,0,0
	};	

	int n_PixelFormat = ::ChoosePixelFormat(hDC,&pixelDesc);
	SetPixelFormat(hDC,n_PixelFormat,&pixelDesc);
	hRC = wglCreateContext(hDC);
```	

### fonts
#### bitmap font:
wglUseFontBitmaps function creates a set of bitmap display lists for use in the current OpenGL rendering context. 

generate the font:
```cpp
		SelectObject(hDC,GetStockObject(DEFAULT_GUI_FONT)); //choose different font here!
		wglUseFontBitmaps(hDC,0,256,1000); //256 characters with listbase 1000
```		
display the font:
```cpp
	void GlText(float x, float y, const char *pStr)
	{
		if(strlen(pStr)!=0)
		{
			glListBase(1000);
			glRasterPos2f(x, y); // Position
			glCallLists(strlen(pStr), GL_UNSIGNED_BYTE, pStr);   //Display
		}
	}
```

#### Texture font:
You can download any font bitmap and get those characters from the bitmap
- first load the bitmap and create the texture
- decompose the bmp and get the character and build the display list

```cpp
int glWin::LoadGLTextures(const char* bmpfname, GLuint& texture)                                    // Load Bitmaps And Convert To Textures
{
    int Status=FALSE;                               // Status Indicator
    //AUX_RGBImageRec *TextureImage;               // Create Storage Space For The Textures
	bitmap *TextureImage=new bitmap();
    //if (TextureImage=LoadBMP(bmpfname))
	if(TextureImage->LoadBMP(bmpfname))
    {
		Status=TRUE;
        glGenTextures(1, &texture);          // Create Two Texture
		
	    glBindTexture(GL_TEXTURE_2D, texture);
		glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_MAG_FILTER,GL_LINEAR);
		glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_MIN_FILTER,GL_LINEAR);
		glTexImage2D(GL_TEXTURE_2D, 0, 3, TextureImage->nx, TextureImage->ny, 0, GL_RGB, GL_UNSIGNED_BYTE, TextureImage->data);
    }
	delete TextureImage;
    return Status;                                  // Return The Status
}

GLvoid glWin::BuildFont(GLvoid)								// Build Our Font Display List
{
	float	cx;											// Holds Our X Character Coord
	float	cy;											// Holds Our Y Character Coord

	base=glGenLists(256);								// Creating 256 Display Lists
	glBindTexture(GL_TEXTURE_2D, texture_font);			// Select Our Font Texture
	for (int loop=0; loop<256; loop++)						// Loop Through All 256 Lists
	{
		cx=float(loop%16)/16.0f;						// X Position Of Current Character
		cy=float(loop/16)/16.0f;						// Y Position Of Current Character

		glNewList(base+loop,GL_COMPILE);				// Start Building A List
			glBegin(GL_QUADS);							// Use A Quad For Each Character
				glTexCoord2f(cx,1-cy-0.0625f);			// Texture Coord (Bottom Left)
				glVertex2i(0,0);						// Vertex Coord (Bottom Left)
				glTexCoord2f(cx+0.0625f,1-cy-0.0625f);	// Texture Coord (Bottom Right)
				glVertex2i(14,0);						// Vertex Coord (Bottom Right)
				glTexCoord2f(cx+0.0625f,1-cy);			// Texture Coord (Top Right)
				glVertex2i(14,14);						// Vertex Coord (Top Right)
				glTexCoord2f(cx,1-cy);					// Texture Coord (Top Left)
				glVertex2i(0,14);						// Vertex Coord (Top Left)
			glEnd();									// Done Building Our Quad (Character)
			glTranslated(12,0,0);						// Move To The Right Of The Character
		glEndList();									// Done Building The Display List
	}													// Loop Until All 256 Are Built
}
```

use the texture font:
```cpp
GLvoid glWin::glPrint(float glx, float gly, const char *string, int set,float sx,float sy,float shx,float shy)	// Where The Printing Happens
{
	if (set>1)
	{
		set=1;
	}

	int x,y;
	
	gl2win(glx,gly,&x,&y,sx,sy,shx,shy);
	//y=glWinRect.bottom-y; //note opengl is up coordinate
	glEnable(GL_TEXTURE_2D);
	glBindTexture(GL_TEXTURE_2D, texture_font);			// Select Our Font Texture
	glDisable(GL_DEPTH_TEST);							// Disables Depth Testing
	
	glMatrixMode(GL_PROJECTION);						// Select The Projection Matrix
	glPushMatrix();										// Store The Projection Matrix
	glLoadIdentity();									// Reset The Projection Matrix
	glOrtho(0,glWinRect.right,0,glWinRect.bottom,-1,1);							// Set Up An Ortho Screen
	glMatrixMode(GL_MODELVIEW);							// Select The Modelview Matrix
	glPushMatrix();										// Store The Modelview Matrix
	glLoadIdentity();									// Reset The Modelview Matrix
	
	glTranslated(x,glWinRect.bottom-y,0);								// Position The Text (0,0 - Bottom Left)
	//glRasterPos2f(x, glWinRect.bottom-y);
	//glScalef(scalex, scaley, 1.0);
	glListBase(base-32+(128*set));						// Choose The Font Set (0 or 1) 32 is the char space
	glCallLists(strlen(string),GL_UNSIGNED_BYTE,string);// Write The Text To The Screen
	
	glMatrixMode(GL_PROJECTION);						// Select The Projection Matrix
	glPopMatrix();										// Restore The Old Projection Matrix
	glMatrixMode(GL_MODELVIEW);							// Select The Modelview Matrix
	glPopMatrix();										// Restore The Old Projection Matrix
	
	glEnable(GL_DEPTH_TEST);							// Enables Depth Testing
	glDisable(GL_TEXTURE_2D);
}
```

#### outline font
outline font: it gives the outline for the character.
also you can download a lot of outline fonts.

```cpp
class COutlineFont 
{	
public:
	COutlineFont(CDC* dc, char* fontname);
	virtual ~COutlineFont();
	
	void TextDisplay(POINT pt,double theta,CString text,float color_r,float color_g,float color_b, float scale);
	void TextDisplay(POINT pt,double theta,CString text,float color_r,float color_g,float color_b, float scaleX,float scaleY);
	void DrawString(char* s); 
	void Text(float x,float y,const char *str,float scalex,float scaley);
	void Text(float x,float y,float z, const char *str,float scalex,float scaley,float scalez);
private:
	GLuint m_listbase;
	CDC* m_pDC;
	
private:
	// Hide these.
	COutlineFont() { }
	COutlineFont(const COutlineFont& obj) { }
	COutlineFont& operator=(const COutlineFont& obj) { return *this; }
};
```

to generate the outline font and dislay them

```cpp
#include "StdAfx.h"
#include "afxwin.h"
#include "OutlineFont.h"

#pragma warning(disable:4244)

COutlineFont::COutlineFont(
						   CDC* dc, 
						   char* fontname)
{
	// Class constructor.
	// Stores each character in its own display list
	// for later drawing via the wglUseFontOutlines() call.

	if (dc && fontname && strlen(fontname) > 0) {

		m_pDC = dc;
		m_listbase = glGenLists(256);

		//GL_INVALID_VALUE 
		//GL_INVALID_OPERATION  

		// Setup the Font characteristics
		LOGFONT logfont;
		GLYPHMETRICSFLOAT gmf[256];
		
		logfont.lfHeight        = -12;
		logfont.lfWidth         = 0;
		logfont.lfEscapement    = 0;
		logfont.lfOrientation   = logfont.lfEscapement;
		logfont.lfWeight        = FW_THIN;
		logfont.lfItalic        = FALSE;
		logfont.lfUnderline     = FALSE;
		logfont.lfStrikeOut     = FALSE;
		logfont.lfCharSet       = ANSI_CHARSET;
		logfont.lfOutPrecision  = OUT_TT_PRECIS;
		logfont.lfClipPrecision = CLIP_DEFAULT_PRECIS;
		logfont.lfQuality       = ANTIALIASED_QUALITY;
		logfont.lfPitchAndFamily = FF_ROMAN|DEFAULT_PITCH;
		strcpy(logfont.lfFaceName, fontname);

		CFont font;
		CFont* oldfont;
		BOOL success = font.CreateFontIndirect(&logfont);
		oldfont = m_pDC->SelectObject(&font);
		if (!success || 
			FALSE == wglUseFontOutlines(
			m_pDC->m_hDC, 
			0, 
			256, 
			m_listbase,
			0.0f, // Deviation From The True Outlines
			0.2f, // Font Thickness In The Z Direction
			WGL_FONT_POLYGONS,
			gmf)) 
		{
			glDeleteLists(m_listbase, 256);
			m_listbase = 0;
		}
		else 
		{
			m_pDC->SelectObject(oldfont);
		}
	}
}

COutlineFont::~COutlineFont()
{
	// Class destructor.

	glDeleteLists(m_listbase, 256);
	m_listbase = 0;
}


void COutlineFont::DrawString(char* s)
{
	// Draws the given text string.

	GLsizei len = GLsizei(strlen(s));
	if (s && len > 0) {
		// Must save/restore the list base.
		glPushAttrib(GL_LIST_BIT);{
			glListBase(m_listbase);
			glCallLists(len, GL_UNSIGNED_BYTE, (const GLvoid*)s);
		} glPopAttrib();
	}
}


void COutlineFont::TextDisplay(POINT pt,double theta,CString text,float color_r,float color_g,float color_b, float scale)
{
	
	int sl = text.GetLength();
	
	glColor3f(color_r,color_g,color_b);
	
	glPushMatrix(); 
	
	glTranslatef(pt.x,pt.y, 0.0);
	glRotatef(180.0,180.0, theta, 1.0);
	//glTranslatef(-5*sl,-6, 0.0);
	glTranslatef(-5 * sl, -8, 0.0);
	glScalef(scale, scale, scale);
	
	DrawString(text.GetBuffer(sl));
	
	glPopMatrix();

}

void COutlineFont::TextDisplay(POINT pt,double theta,CString text,float color_r,float color_g,float color_b, float scaleX,float scaleY)
{
	
	int sl = text.GetLength();
	
	glColor3f(color_r,color_g,color_b);
	
	glPushMatrix(); 
	
	glTranslatef(pt.x,pt.y, 0.0);
	glRotatef(180.0,180.0, theta, 1.0);
	//glTranslatef(-5*sl,-6, 0.0);
	//glTranslatef(-5 * sl, -8, 0.0);
	glScalef(scaleX, scaleY, scaleX);
	
	DrawString(text.GetBuffer(sl));
	
	glPopMatrix();

}

void COutlineFont::Text(float x,float y,const char *str,float scalex,float scaley)
{
	int len=strlen(str);
	if(len)
	{
		glPushMatrix();
		//glLoadIdentity();
		glTranslatef(x,y,0.0);
		//glRotatef(180.0,180.0, 0, 1.0);
		glScalef(scalex, 1.0*scaley, 1.0);
		glPushAttrib(GL_LIST_BIT);
		glListBase(m_listbase);
		glCallLists(len, GL_UNSIGNED_BYTE, (const GLvoid*)str);
		glPopAttrib();
		
		glPopMatrix();
	}
}

void COutlineFont::Text(float x,float y,float z,const char *str,float scalex,float scaley,float scalez)
{
	int len=strlen(str);
	if(len)
	{
		glPushMatrix();
		//glLoadIdentity();
		glTranslatef(x,y,z);
		//glRotatef(180.0,180.0, 0, 1.0);
		glScalef(scalex, scaley, scalez);
		glPushAttrib(GL_LIST_BIT);
		glListBase(m_listbase);
		glCallLists(len, GL_UNSIGNED_BYTE, (const GLvoid*)str);
		glPopAttrib();
		
		glPopMatrix();
	}
}
```
Outline font can use the windows installed fonts.

new COutLineFont(pDC,"Tahoma Bold")

ONLY after the opengl context is setup, opengl functions can work then.

### setup gl mapping and viewing
set viewport
set matrix mode
set coordinate system

a window with 2d opengl:
```cpp
void glWin::setup_RC(int left_off,int right_off)
{
	float w,h;
	//float xmin,xmax,ymin,ymax;

	wglMakeCurrent(hDC,hRC);

	GetClientRect(&client_area);	
	glWinRect=client_area;
	glWinRect.left+=left_off;
	glWinRect.right-=right_off;
	w = glWinRect.right  - glWinRect.left;//-left_off-right_off;
	h = glWinRect.bottom - glWinRect.top;
	
	glViewport(glWinRect.top,glWinRect.left+left_off,w,h);
	
	
	glMatrixMode(GL_PROJECTION);
	glLoadIdentity();
	
	aspectRatio = (GLfloat)w / (GLfloat)h;
	
	//make the origin in the center is good for flip!
	if (w <= h) 
	{
		ogl_xmin=-25;
		ogl_xmax=25;
		ogl_ymin=-25.0 / aspectRatio;
		ogl_ymax=25.0 / aspectRatio;
		num_pixel_per_ogl=w/50.0;
		glOrtho(ogl_xmin,ogl_xmax,ogl_ymin,ogl_ymax,-1,1);
	}
	else
	{
		ogl_xmin=-25.0 * aspectRatio;
		ogl_xmax=25.0 * aspectRatio;
		ogl_ymin=-25.0;
		ogl_ymax=25;
		num_pixel_per_ogl=h/50.0;
		//glOrtho (-25.0 * aspectRatio, 25.0 * aspectRatio, -50.0, 0.0, -50.0, 50.0);
		glOrtho(ogl_xmin,ogl_xmax,ogl_ymin,ogl_ymax,-1,1);
	}

	font_scale=16.0/num_pixel_per_ogl;
	glMatrixMode(GL_MODELVIEW);
	glLoadIdentity();		
	

	wglMakeCurrent(NULL,NULL);
}
```

with viewport and mapping, you can use mixed opengl and wingdi (buttons et al)

### using OpenGL advance features
Windows standared opengl support only OpenGL 1.1. To use more advanced features, you shall include glext.h

often used buffer objects:

```cpp
namespace opengl
{
#ifdef _WIN32
	//VBO supporting & PBO uses the same stuff
	PFNGLGENBUFFERSARBPROC pglGenBuffersARB = 0;                     // VBO Name Generation Procedure
	PFNGLBINDBUFFERARBPROC pglBindBufferARB = 0;                     // VBO Bind Procedure
	PFNGLBUFFERDATAARBPROC pglBufferDataARB = 0;                     // VBO Data Loading Procedure
	PFNGLBUFFERSUBDATAARBPROC pglBufferSubDataARB = 0;               // VBO Sub Data Loading Procedure
	PFNGLDELETEBUFFERSARBPROC pglDeleteBuffersARB = 0;               // VBO Deletion Procedure
	PFNGLGETBUFFERPARAMETERIVARBPROC pglGetBufferParameterivARB = 0; // return various parameters of VBO
	PFNGLMAPBUFFERARBPROC pglMapBufferARB = 0;                       // map VBO procedure
	PFNGLUNMAPBUFFERARBPROC pglUnmapBufferARB = 0;                   // unmap VBO procedure
	
	#define glGenBuffersARB           pglGenBuffersARB
	#define glBindBufferARB           pglBindBufferARB
	#define glBufferDataARB           pglBufferDataARB
	#define glBufferSubDataARB        pglBufferSubDataARB
	#define glDeleteBuffersARB        pglDeleteBuffersARB
	#define glGetBufferParameterivARB pglGetBufferParameterivARB
	#define glMapBufferARB            pglMapBufferARB
	#define glUnmapBufferARB          pglUnmapBufferARB

	// Framebuffer object
	PFNGLGENFRAMEBUFFERSPROC                     pglGenFramebuffers = 0;                      // FBO name generation procedure
	PFNGLDELETEFRAMEBUFFERSPROC                  pglDeleteFramebuffers = 0;                   // FBO deletion procedure
	PFNGLBINDFRAMEBUFFERPROC                     pglBindFramebuffer = 0;                      // FBO bind procedure
	PFNGLCHECKFRAMEBUFFERSTATUSPROC              pglCheckFramebufferStatus = 0;               // FBO completeness test procedure
	PFNGLGETFRAMEBUFFERATTACHMENTPARAMETERIVPROC pglGetFramebufferAttachmentParameteriv = 0;  // return various FBO parameters
	PFNGLGENERATEMIPMAPPROC                      pglGenerateMipmap = 0;                       // FBO automatic mipmap generation procedure
	PFNGLFRAMEBUFFERTEXTURE2DPROC                pglFramebufferTexture2D = 0;                 // FBO texdture attachement procedure
	PFNGLFRAMEBUFFERRENDERBUFFERPROC             pglFramebufferRenderbuffer = 0;              // FBO renderbuffer attachement procedure
	// Renderbuffer object
	PFNGLGENRENDERBUFFERSPROC                    pglGenRenderbuffers = 0;                     // renderbuffer generation procedure
	PFNGLDELETERENDERBUFFERSPROC                 pglDeleteRenderbuffers = 0;                  // renderbuffer deletion procedure
	PFNGLBINDRENDERBUFFERPROC                    pglBindRenderbuffer = 0;                     // renderbuffer bind procedure
	PFNGLRENDERBUFFERSTORAGEPROC                 pglRenderbufferStorage = 0;                  // renderbuffer memory allocation procedure
	PFNGLGETRENDERBUFFERPARAMETERIVPROC          pglGetRenderbufferParameteriv = 0;           // return various renderbuffer parameters
	PFNGLISRENDERBUFFERPROC                      pglIsRenderbuffer = 0;                       // determine renderbuffer object type

	#define glGenFramebuffers                        pglGenFramebuffers
	#define glDeleteFramebuffers                     pglDeleteFramebuffers
	#define glBindFramebuffer                        pglBindFramebuffer
	#define glCheckFramebufferStatus                 pglCheckFramebufferStatus
	#define glGetFramebufferAttachmentParameteriv    pglGetFramebufferAttachmentParameteriv
	#define glGenerateMipmap                         pglGenerateMipmap
	#define glFramebufferTexture2D                   pglFramebufferTexture2D
	#define glFramebufferRenderbuffer                pglFramebufferRenderbuffer

	#define glGenRenderbuffers                       pglGenRenderbuffers
	#define glDeleteRenderbuffers                    pglDeleteRenderbuffers
	#define glBindRenderbuffer                       pglBindRenderbuffer
	#define glRenderbufferStorage                    pglRenderbufferStorage
	#define glGetRenderbufferParameteriv             pglGetRenderbufferParameteriv
	#define glIsRenderbuffer                         pglIsRenderbuffer


#endif
	PFNWGLEXTSWAPCONTROLPROC wglSwapIntervalEXT = NULL;
	PFNWGLEXTGETSWAPINTERVALPROC wglGetSwapIntervalEXT = NULL;
	
	//init VSync func
	bool InitVSync()
	{
		//get extensions of graphics card
		char* extensions = (char*)glGetString(GL_EXTENSIONS);
		
		//is WGL_EXT_swap_control in the string? VSync switch possible?
		if (strstr(extensions,"WGL_EXT_swap_control"))
		{
			//get address's of both functions and save them
			wglSwapIntervalEXT = (PFNWGLEXTSWAPCONTROLPROC)
				wglGetProcAddress("wglSwapIntervalEXT");
			wglGetSwapIntervalEXT = (PFNWGLEXTGETSWAPINTERVALPROC)
				wglGetProcAddress("wglGetSwapIntervalEXT");
			return true;
		}
		else
			return false;	
	}
	
	bool VSyncEnabled()
	{
		//if interval is positive, it is not 0 so enabled ;)
		return (wglGetSwapIntervalEXT()> 0);
	}
	
	void SetVSyncState(bool enable) //this is the only interface to user.
	{
		if(InitVSync())
		{
			if (enable)	wglSwapIntervalEXT(1); //set interval to 1 -&gt; enable
			else wglSwapIntervalEXT(0); //disable
		}
	}
	
	bool is_vbo_supported()
	{
		glGenBuffersARB = (PFNGLGENBUFFERSPROC)wglGetProcAddress("glGenBuffersARB");
		glBindBufferARB = (PFNGLBINDBUFFERPROC)wglGetProcAddress("glBindBufferARB");
		glBufferDataARB = (PFNGLBUFFERDATAPROC)wglGetProcAddress("glBufferDataARB");
		glBufferSubDataARB = (PFNGLBUFFERSUBDATAPROC)wglGetProcAddress("glBufferSubDataARB");
		glDeleteBuffersARB = (PFNGLDELETEBUFFERSPROC)wglGetProcAddress("glDeleteBuffersARB");
		glGetBufferParameterivARB = (PFNGLGETBUFFERPARAMETERIVPROC)wglGetProcAddress("glGetBufferParameterivARB");
		glMapBufferARB = (PFNGLMAPBUFFERPROC)wglGetProcAddress("glMapBufferARB");
		glUnmapBufferARB = (PFNGLUNMAPBUFFERPROC)wglGetProcAddress("glUnmapBufferARB");
		glGenerateMipmap = (PFNGLGENERATEMIPMAPPROC)wglGetProcAddress("glGenerateMipmap");
		// check once again VBO extension
		if(glGenBuffersARB && glBindBufferARB && glBufferDataARB && glBufferSubDataARB &&
			glMapBufferARB && glUnmapBufferARB && glDeleteBuffersARB && glGetBufferParameterivARB)
		{
			//vboSupported = vboUsed = true;
			//cout << "Video card supports GL__vertex_buffer_object." << endl;
			printf("VBO is supported\n");
			return true;
		}
		else
		{
			//vboSupported = vboUsed = false;
			//cout << "Video card does NOT support GL__vertex_buffer_object." << endl;
			printf("VBO not supported\n");
			return false;
		}
	}

	void GlBlender(bool enable)
	{
		if(enable)
		{
			glEnable(GL_BLEND);
			glEnable(GL_POINT_SMOOTH);
			glHint(GL_POINT_SMOOTH_HINT,GL_NICEST);
		//	glEnable(GL_LINE_SMOOTH);
		//	glHint(GL_LINE_SMOOTH_HINT,GL_NICEST);
			glEnable(GL_POLYGON_SMOOTH);
			glHint(GL_POLYGON_SMOOTH_HINT,GL_NICEST);
		}
		else
		{
			glDisable(GL_BLEND);
			glDisable(GL_POINT_SMOOTH);
			glDisable(GL_LINE_SMOOTH);
			glDisable(GL_POLYGON_SMOOTH);
		}
	}

	void GlText(float x, float y, const char *pStr)
	{
		if(strlen(pStr)!=0)
		{
			//glPushAttrib();
			//glPushMatrix();
			glListBase(1000);
			//glColor3f(1,1,0);
			glRasterPos2f(x, y); // Position
			glCallLists(strlen(pStr), GL_UNSIGNED_BYTE, pStr);   //Display
			//glPopAttrib();
			//glPopMatrix();
		}
	}

	void GlTextInit(HDC hDC)
	{
		wglUseFontBitmaps(hDC, 0, 128, 1000); // hDC = wglGetCurrentDC()
	}

	BOOL SetupPixelFormat(HDC hDC)
	{
		PIXELFORMATDESCRIPTOR pixelDesc=
		{
			sizeof(PIXELFORMATDESCRIPTOR),
				1,
				PFD_DRAW_TO_WINDOW|PFD_SUPPORT_OPENGL|
				PFD_DOUBLEBUFFER,
				PFD_TYPE_RGBA,
				24,
				0,0,0,0,0,0,
				0,
				0,
				0,
				0,0,0,0,
				32,
				0,
				0,
				PFD_MAIN_PLANE,
				0,
				0,0,0
		};
		
		int pixelformat;
		
		if ( (pixelformat = ChoosePixelFormat(hDC, &pixelDesc)) == 0 )
		{
 			MessageBox(NULL, "ChoosePixelFormat failed", "Error", MB_OK);
			return FALSE;
		}
		
		if (SetPixelFormat(hDC, pixelformat, &pixelDesc) == FALSE)
		//if (SetPixelFormat(hDC, 6, &pixelDesc) == FALSE)
		//if (SetPixelFormat(hDC, 9, &pixelDesc) == FALSE)
		{
			MessageBox(NULL, "SetPixelFormat failed", "Error", MB_OK);
			return FALSE;
		}
		return TRUE;
	}

	BOOL SetupPixelFormat(HDC hDC, int mode)
	{
		PIXELFORMATDESCRIPTOR pixelDesc=
		{
			sizeof(PIXELFORMATDESCRIPTOR),
				1,
				PFD_DRAW_TO_WINDOW|PFD_SUPPORT_OPENGL|
				PFD_DOUBLEBUFFER,
				PFD_TYPE_RGBA,
				24,
				0,0,0,0,0,0,
				0,
				0,
				0,
				0,0,0,0,
				32,
				0,
				0,
				PFD_MAIN_PLANE,
				0,
				0,0,0
		};
		
		int pixelformat;
		
		if ( (pixelformat = ChoosePixelFormat(hDC, &pixelDesc)) == 0 )
		{
			MessageBox(NULL, "ChoosePixelFormat failed", "Error", MB_OK);
			return FALSE;
		}
		
		if (SetPixelFormat(hDC, mode, &pixelDesc) == FALSE)
		{
			MessageBox(NULL, "SetPixelFormat failed", "Error", MB_OK);
			return FALSE;
		}
		return TRUE;
	}

}
```

### modelview and matrix
OpenGL uses 4 dimensional space conversion which is necessary.

model: construct the object in its own coordinate
view: move/rotate the object to its position. uaing translate and scale, rotate.

### a glWin abstract class for openGL window
A openGL window includes:
- opengl functions
- setup gl environments
- mappings
- coordinate conversion
- data driven the graphics updates
- metric
- fonts

It is also a CWnd, so CWnd functions can be used too.

```cpp
#pragma once
//this is a type of window rendered by opengl
//it may cover BDwin, spectrum, M mode
//#include "gl\glaux.h"		// Header File For The Glaux Library
//glaux.lib deprecated, replaced it with my own code
#include "outlinefont.h"
#include <list>
#include "AnnotationBox.h"

class textNotation
{
public:
	float left, bottom;       //phy coordinates of the bottom-left corner of the textBox
	std::string str;
	int AffiliatedWindowID;
	bool bIsSelected;
	bool bIsMeasurement;      //measurement relevant nodes

public:
	textNotation() 
	{ 
		left = 0;
		bottom = 0;
		str.clear(); 
		AffiliatedWindowID = -1;
		bIsSelected = 0;
		bIsMeasurement = 0;
	}
};

class arrowMark
{
public:
	float phy_x, phy_y;   //phy coordinates
	float ang;            //in radian
	CWnd* affiliatedWindow;

public:
	arrowMark()
	{
		phy_x = 0.0f;
		phy_y = 0.0f;
		ang = 0.0f;
		affiliatedWindow = NULL;
	}
};

class glWin:public CWnd
{
public:
	enum ogl_font {BitmapFont,TextureFont,OutlineFont};

protected:
	HGLRC hRC;
	HDC hDC;
	RECT client_area;

	COutlineFont *pText;
	CDC *pDC;
	bool use_outlinefont;
	ogl_font font_selected;
	GLuint texture_font;
	GLuint	base; //display list base for texture

	RECT glWinRect;

	float shiftx,shifty;
	float scalex,scaley;
	float resol_x,resol_y;
	float aspectRatio;

	float ogl_xmin,ogl_xmax,ogl_ymin,ogl_ymax;
	float num_pixel_per_ogl;
	bool keep_same_fontsize;
	float font_scale; //to use the font to scale some elements' coordinate
	int flip_lr,flip_ud;
	//float max_ogl_range; //maximum size of opengl range

	std::list<textNotation> annotationList;
	textNotation currentAnnotation;
	CAnnotationBox m_Edit_Annotation;
	CFont* annotationFont;
	float fAnotationDragPtPhyX, fAnotationDragPtPhyY;
	CPoint cpAnotationDragPtWin;
	
	GLuint arrowTexture, arrowTexture2;	   //arrowTexture2 is the blue/active arrow
	arrowMark currentArrowMark;	
	std::list<CPoint> prevCursorPts;
	bool bNeedGenGLRC;
public:
	glWin();
	virtual ~glWin(); //need to be virtual to call derivative

	void clear();
	virtual void update_data(int data_type)=0; //may have different implementation
	void setup_ogl(HGLRC rcshare=0);
	virtual void RenderScene()=0; //may have different implementation
	void glText(float x,float y,const char *s,float scalex=1.0,float scaley=1.0, float scale = 1.0f,float shx=0,float shy=0);
	void setup_RC(int left_off=0,int right_off=0);
	void set_shift(float dx,float dy) {shiftx=dx;shifty=dy;}
	void set_scale(float sx,float sy) {scalex=sx;scaley=sy;}
	void set_shiftx(float dx) {shiftx=dx;}
	void set_shifty(float dy) {shifty=dy;}
	void set_scalex(float sx) {scalex=sx;}
	void set_scaley(float sy) {scaley=sy;}
	void get_ogl_rgn(float *rgn); 
	float get_shiftx() const {return shiftx;}
	float get_shifty() const {return shifty;}
	float get_scalex() const {return scalex;}
	float get_scaley() const {return scaley;}
	RECT get_glWinRect() const {return glWinRect;}
	HGLRC get_rc() const {return hRC;}
	//zoom operation
	void shift(float dx,float dy) {shiftx+=dx;shifty+=dy;}
	void scale(float sx,float sy) {scalex*=sx;scaley*=sy;}
	float get_pixel_per_ogl() const {return num_pixel_per_ogl;}

	void fliplr(){flip_lr*=-1;scalex*=-1;}
	void flipud(){flip_ud*=-1;scaley*=-1;}
	int get_fliplr() const {return flip_lr;}
	int get_flipud() const {return flip_ud;}
	float get_resol_x(); //calculate the pixel length in mm
	float get_resol_y();
	COutlineFont* get_ptext() {return pText;}
	void gl2win(float glx,float gly,int *x,int *y,float sx,float sy,float shx,float shy);	
	void win2gl(int winx, int winy, float *glx, float *gly);
	void choose_font(ogl_font fnt) {if(fnt>=BitmapFont &&fnt<=OutlineFont) font_selected=fnt;}
	void release();

	void SetAnnotationBoxInFoxus();
	CString GetCurrentAnnotation();
	void ClearCurrentAnnotation();
	void DeleteLastAnnotation(int count);
	void DeleteAllAnnotations();
	void ClearMeasurementAnnotation();
	void ConfirmAnnotation() {m_Edit_Annotation.ConfirmAnnotation();}
	virtual void SaveAnnotation(std::string str, float left, float bottom) = 0;
	void AppendAnnotation(const char* str);

	void DeleteAllArrowMarks();


protected:
	virtual void phy2gl(float phy_x, float phy_y, float* glx, float* gly) = 0;
	void DrawArrowSeries();
	void DrawArrow(float x, float y, float ang, float size);
	int LoadGLTextures(const char *bmpfname,GLuint& texture);
	int LoadGlTextureswAlpha(const char *bmpfname,GLuint& texture);

	//these are for 2D texture fonts
	//AUX_RGBImageRec *LoadBMP(const char *Filename);                // Loads A Bitmap Image
	int LoadGLTextures();                                    // Load Bitmaps And Convert To Textures
	GLvoid BuildFont(GLvoid);								// Build Our Font Display List
	GLvoid KillFont(GLvoid);									// Delete The Font From Memory
	GLvoid glPrint(float glx, float gly, const char *string, int set,float sx=1,float sy=1,float shx=0,float shy=0);	// Where The Printing Happens

public:
	//void set_max_ogl_range(float r) {max_ogl_range=r;}
	//const float get_ogl_range() const {return max_ogl_range;}
	//{{AFX_VIRTUAL(glWin)
	virtual void DoDataExchange(CDataExchange* pDX);
	//}}AFX_VIRTUAL(glWin)
	
	// Generated message map functions
	//{{AFX_MSG(glWin)
	afx_msg int OnCreate(LPCREATESTRUCT lpCreateStruct);
	afx_msg void OnMouseMove(UINT nFlags, CPoint point);
	//}}AFX_MSG
	DECLARE_MESSAGE_MAP()	
};

```

Implementations:

```cpp
#include "StdAfx.h"

#include <afxwin.h>
#include "opengl.h"
#include "glwin.h"
#include "bitmap.h"
#include "resource.h"

using namespace opengl;

#pragma warning(disable:4244)
//#pragma comment(lib,"glaux.lib")
 
//#define IDC_EDIT_ANNOTATION 0x2000

void glWin::clear()
{
	glClearColor(0.0f,0.0f,0.0f,1.0f);
	glClearStencil(0.0f);
	glClear(GL_COLOR_BUFFER_BIT|GL_DEPTH_BUFFER_BIT|GL_STENCIL_BUFFER_BIT);
}

void glWin::clear(float r,float g,float b)
{
	glClearColor(r, g, b, 1.0f);
	glClearStencil(0.0f);
	glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT | GL_STENCIL_BUFFER_BIT);
}

void glWin::setup_ogl(HGLRC rcshare)
{
	hDC=::GetDC(m_hWnd);

	PIXELFORMATDESCRIPTOR pixelDesc =
	{
		sizeof(PIXELFORMATDESCRIPTOR),
			1,
			PFD_DRAW_TO_WINDOW|PFD_SUPPORT_OPENGL|
			PFD_DOUBLEBUFFER,
			PFD_TYPE_RGBA,
			24,
			0,0,0,0,0,0,
			0,
			0,
			0,
			0,0,0,0,
			32,//
			0,
			0,
			PFD_MAIN_PLANE,
			0,
			0,0,0
	};	

#ifdef AMD_GRAPHIC_CARD
	int n_PixelFormat = 14;
#else
	int n_PixelFormat = ::ChoosePixelFormat(hDC,&pixelDesc);
#endif	
	SetPixelFormat(hDC,n_PixelFormat,&pixelDesc);

	if(rcshare) hRC=rcshare;
	else hRC = wglCreateContext(hDC);


	wglMakeCurrent(hDC,hRC);
	opengl::is_vbo_supported();
	opengl::SetVSyncState(0);
	switch(font_selected)
	{
	case BitmapFont:
		SelectObject(hDC,GetStockObject(DEFAULT_GUI_FONT)); //choose different font here!
		wglUseFontBitmaps(hDC,0,256,1000); //256 characters with listbase 1000
		break;
	case TextureFont:
		LoadGLTextures();
		BuildFont();
		break;
	case OutlineFont:
		if(!pDC && !pText) //for repeatedly call setup_RC memory leak
		{
			pDC=new CDC();
			pDC->Attach(hDC);
			pText=new COutlineFont(pDC,"Tahoma Bold");
		}
		break;
	}
	/*
	if(!use_outlinefont)
	{
		SelectObject(hDC,GetStockObject(DEFAULT_GUI_FONT)); //choose different font here!
		wglUseFontBitmaps(hDC,0,256,1000); //256 characters with listbase 1000
		//glText=glText;
	}
	else
	{
	//use outline font!
		if(!pDC && !pText) //for repeatedly call setup_RC memory leak
		{
			pDC=new CDC();
			pDC->Attach(hDC);
			//pText=new COutlineFont(pDC,"Tahoma");
			//pText=new COutlineFont(pDC,"Helvetica");
			pText=new COutlineFont(pDC,"Arial");
		}
		//glText=COutlineFont::Text;
	}
	*/
	glDisable(GL_ALPHA_TEST);
	glDisable(GL_DEPTH_TEST);
	glFrontFace(GL_CW);
	glShadeModel(GL_SMOOTH);
	glPolygonMode(GL_FRONT_AND_BACK,GL_FILL);

	wglMakeCurrent(NULL, NULL); 
}

glWin::glWin()
{
	pDC=0;
	pText=0;
	keep_same_fontsize=1;
	font_selected = OutlineFont;//BitmapFont;//OutlineFont;
	base=0;
	aspectRatio=0.0f;
	bNeedGenGLRC=1;
}

glWin::~glWin()
{
	release();
}

void glWin::release()
{
	delete pDC;pDC=0;
	delete pText;pText=0;
	if(base) 
	{
		KillFont();
		base=0;
	}
}
void glWin::glText(float x,float y,const char *s,float scalex,float scaley, float scale,float shx,float shy)
{
/*	if(use_outlinefont)
	{
		float sc=1.0;
		if(keep_same_fontsize)
		{
			sc=16*0.75/num_pixel_per_ogl;
		}
		pText->Text(x,y,s,sc*scale/(scalex*0.75),sc*scale/(scaley*0.75));
	}
	else
	{
		opengl::GlText(x,y,s);
	}
	*/
	float sc=1.0;
	if(keep_same_fontsize)
	{
		sc=16*0.75/num_pixel_per_ogl;
	}

	switch(font_selected)
	{
	case OutlineFont:
		pText->Text(x,y,s,sc*scale/(scalex*0.75),sc*scale/(scaley*0.75));
		break;
	case BitmapFont:
		//glPushMatrix();
		//glLoadIdentity();
		//glScalef(scalex, scaley, 1.0);
		opengl::GlText(x,y,s);
		//glPopMatrix();
		break;
	case TextureFont:
		//glPrint(x,y,s,0,sc*scale/(scalex*0.75),sc*scale/(scaley*0.75),shx,shy);
		glPrint(x,y,s,0,scalex,scaley,shx,shy);
		break;
	}
}

void glWin::setup_RC(int left_off,int right_off)
{
	float w,h;
	//float xmin,xmax,ymin,ymax;

	wglMakeCurrent(hDC,hRC);

	GetClientRect(&client_area);	
	glWinRect=client_area;
	glWinRect.left+=left_off;
	glWinRect.right-=right_off;
	w = glWinRect.right  - glWinRect.left;//-left_off-right_off;
	h = glWinRect.bottom - glWinRect.top;
	
	glViewport(glWinRect.top,glWinRect.left+left_off,w,h);
	
	
	glMatrixMode(GL_PROJECTION);
	glLoadIdentity();
	
	aspectRatio = (GLfloat)w / (GLfloat)h;
	
	//make the origin in the center is good for flip!
	if (w <= h) 
	{
		ogl_xmin=-25;
		ogl_xmax=25;
		ogl_ymin=-25.0 / aspectRatio;
		ogl_ymax=25.0 / aspectRatio;
		num_pixel_per_ogl=w/50.0;
		glOrtho(ogl_xmin,ogl_xmax,ogl_ymin,ogl_ymax,-1,1);
	}
	else
	{
		ogl_xmin=-25.0 * aspectRatio;
		ogl_xmax=25.0 * aspectRatio;
		ogl_ymin=-25.0;
		ogl_ymax=25;
		num_pixel_per_ogl=h/50.0;
		//glOrtho (-25.0 * aspectRatio, 25.0 * aspectRatio, -50.0, 0.0, -50.0, 50.0);
		glOrtho(ogl_xmin,ogl_xmax,ogl_ymin,ogl_ymax,-1,1);
	}

	font_scale=16.0/num_pixel_per_ogl;
	glMatrixMode(GL_MODELVIEW);
	glLoadIdentity();		
	

	wglMakeCurrent(NULL,NULL);
}

void glWin::DoDataExchange(CDataExchange* pDX)
{
	CWnd::DoDataExchange(pDX);
	//{{AFX_DATA_MAP(glWin)
//	DDX_Control(pDX, IDC_EDIT_ANNOTATION, m_Edit_Annotation);
	//}}AFX_DATA_MAP

}

void glWin::get_ogl_rgn(float *rgn)
{
	rgn[0]=ogl_xmin;
	rgn[1]=ogl_xmax;
	rgn[2]=ogl_ymin;
	rgn[3]=ogl_ymax;
}

float glWin::get_resol_x()
{
	return resol_x=(ogl_xmax-ogl_xmin)/(glWinRect.right-glWinRect.left)/scalex;
}

float glWin::get_resol_y()
{
	return resol_y=(ogl_ymax-ogl_ymin)/(glWinRect.bottom-glWinRect.top)/scaley;
}

BEGIN_MESSAGE_MAP(glWin, CWnd)
//{{AFX_MSG_MAP(glWin)
	ON_WM_CREATE()
	//}}AFX_MSG_MAP
	ON_WM_MOUSEMOVE()
END_MESSAGE_MAP()


int glWin::LoadGLTextures()                                    // Load Bitmaps And Convert To Textures
{
    /*int Status=FALSE;                               // Status Indicator
    AUX_RGBImageRec *TextureImage;               // Create Storage Space For The Textures

    if (TextureImage=LoadBMP("Font.bmp"))
    {
		Status=TRUE;
        glGenTextures(1, &texture_font);          // Create Two Texture

	    glBindTexture(GL_TEXTURE_2D, texture_font);
		glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_MAG_FILTER,GL_LINEAR);
		glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_MIN_FILTER,GL_LINEAR);
		glTexImage2D(GL_TEXTURE_2D, 0, 3, TextureImage->sizeX, TextureImage->sizeY, 0, GL_RGB, GL_UNSIGNED_BYTE, TextureImage->data);
    }
	if (TextureImage)							// If Texture Exists
	{
		if (TextureImage->data)			// If Texture Image Exists
		{
			free(TextureImage->data);	// Free The Texture Image Memory
		}
		free(TextureImage);				// Free The Image Structure
	}
    return Status;                                  // Return The Status
	*/
	return LoadGLTextures("font.bmp",texture_font);
}

int glWin::LoadGLTextures(const char* bmpfname, GLuint& texture)                                    // Load Bitmaps And Convert To Textures
{
    int Status=FALSE;                               // Status Indicator
    //AUX_RGBImageRec *TextureImage;               // Create Storage Space For The Textures
	bitmap *TextureImage=new bitmap();
    //if (TextureImage=LoadBMP(bmpfname))
	if(TextureImage->LoadBMP(bmpfname))
    {
		Status=TRUE;
        glGenTextures(1, &texture);          // Create Two Texture
		
	    glBindTexture(GL_TEXTURE_2D, texture);
		glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_MAG_FILTER,GL_LINEAR);
		glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_MIN_FILTER,GL_LINEAR);
		glTexImage2D(GL_TEXTURE_2D, 0, 3, TextureImage->nx, TextureImage->ny, 0, GL_RGB, GL_UNSIGNED_BYTE, TextureImage->data);
    }
	delete TextureImage;
    return Status;                                  // Return The Status
}

int glWin::LoadGlTextureswAlpha( const char *bmpfname, GLuint& texture )
{
	int Status = FALSE;                               // Status Indicator
    //AUX_RGBImageRec *TextureImage;               // Create Storage Space For The Textures
	bitmap *TextureImage=new bitmap();
    //if (TextureImage = LoadBMP(bmpfname))
	if(TextureImage->LoadBMP(bmpfname))
    {
		Status = TRUE;

		unsigned char* TextureImageAlpha = new unsigned char[TextureImage->nx * TextureImage->ny * 4];
		int i = 0;
		while (i < TextureImage->nx * TextureImage->ny)
		{
			TextureImageAlpha[i*4] = TextureImage->data[i*3];
			TextureImageAlpha[i*4+1] = TextureImage->data[i*3+1];
			TextureImageAlpha[i*4+2] = TextureImage->data[i*3+2];
			if (TextureImageAlpha[i*4] + TextureImageAlpha[i*4+1] + TextureImageAlpha[i*4+2] == 0)
				TextureImageAlpha[i*4+3] = 0;
			else
				TextureImageAlpha[i*4+3] = 255;
			i++;
		}

        glGenTextures(1, &texture);          // Create Two Texture		
		glBindTexture(GL_TEXTURE_2D, texture);
		glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_MAG_FILTER,GL_LINEAR);
		glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_MIN_FILTER,GL_LINEAR);
		glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, TextureImage->nx, TextureImage->ny, 0, GL_BGRA, GL_UNSIGNED_BYTE, TextureImageAlpha);

		delete[] TextureImageAlpha;
		
    }
	delete TextureImage;
	return Status;                      // Return The Status
}

GLvoid glWin::BuildFont(GLvoid)								// Build Our Font Display List
{
	float	cx;											// Holds Our X Character Coord
	float	cy;											// Holds Our Y Character Coord

	base=glGenLists(256);								// Creating 256 Display Lists
	glBindTexture(GL_TEXTURE_2D, texture_font);			// Select Our Font Texture
	for (int loop=0; loop<256; loop++)						// Loop Through All 256 Lists
	{
		cx=float(loop%16)/16.0f;						// X Position Of Current Character
		cy=float(loop/16)/16.0f;						// Y Position Of Current Character

		glNewList(base+loop,GL_COMPILE);				// Start Building A List
			glBegin(GL_QUADS);							// Use A Quad For Each Character
				glTexCoord2f(cx,1-cy-0.0625f);			// Texture Coord (Bottom Left)
				glVertex2i(0,0);						// Vertex Coord (Bottom Left)
				glTexCoord2f(cx+0.0625f,1-cy-0.0625f);	// Texture Coord (Bottom Right)
				glVertex2i(14,0);						// Vertex Coord (Bottom Right)
				glTexCoord2f(cx+0.0625f,1-cy);			// Texture Coord (Top Right)
				glVertex2i(14,14);						// Vertex Coord (Top Right)
				glTexCoord2f(cx,1-cy);					// Texture Coord (Top Left)
				glVertex2i(0,14);						// Vertex Coord (Top Left)
			glEnd();									// Done Building Our Quad (Character)
			glTranslated(12,0,0);						// Move To The Right Of The Character
		glEndList();									// Done Building The Display List
	}													// Loop Until All 256 Are Built
}

GLvoid glWin::KillFont(GLvoid)									// Delete The Font From Memory
{
	glDeleteLists(base,256);							// Delete All 256 Display Lists
}

//set=0: non-italic font, set=1: italic font
//note x,y coordinate is in physical coordinate & opengl mixed
GLvoid glWin::glPrint(float glx, float gly, const char *string, int set,float sx,float sy,float shx,float shy)	// Where The Printing Happens
{
	if (set>1)
	{
		set=1;
	}

	int x,y;
	
	gl2win(glx,gly,&x,&y,sx,sy,shx,shy);
	//y=glWinRect.bottom-y; //note opengl is up coordinate
	glEnable(GL_TEXTURE_2D);
	glBindTexture(GL_TEXTURE_2D, texture_font);			// Select Our Font Texture
	glDisable(GL_DEPTH_TEST);							// Disables Depth Testing
	
	glMatrixMode(GL_PROJECTION);						// Select The Projection Matrix
	glPushMatrix();										// Store The Projection Matrix
	glLoadIdentity();									// Reset The Projection Matrix
	glOrtho(0,glWinRect.right,0,glWinRect.bottom,-1,1);							// Set Up An Ortho Screen
	glMatrixMode(GL_MODELVIEW);							// Select The Modelview Matrix
	glPushMatrix();										// Store The Modelview Matrix
	glLoadIdentity();									// Reset The Modelview Matrix
	
	glTranslated(x,glWinRect.bottom-y,0);								// Position The Text (0,0 - Bottom Left)
	//glRasterPos2f(x, glWinRect.bottom-y);
	//glScalef(scalex, scaley, 1.0);
	glListBase(base-32+(128*set));						// Choose The Font Set (0 or 1) 32 is the char space
	glCallLists(strlen(string),GL_UNSIGNED_BYTE,string);// Write The Text To The Screen
	
	glMatrixMode(GL_PROJECTION);						// Select The Projection Matrix
	glPopMatrix();										// Restore The Old Projection Matrix
	glMatrixMode(GL_MODELVIEW);							// Select The Modelview Matrix
	glPopMatrix();										// Restore The Old Projection Matrix
	
	glEnable(GL_DEPTH_TEST);							// Enables Depth Testing
	glDisable(GL_TEXTURE_2D);
}

void glWin::gl2win(float glx,float gly,int *x,int *y,float sx,float sy,float shx,float shy)
{
	glx+=shx;//shiftx+img_shiftx;
	gly+=shy;//shifty;
	glx*=sx;
	gly*=sy;
	//ogl_ymin is the glwinRect.bottom, ogl_ymax is the glWinRect.top
	*x=(glx-ogl_xmin)*(glWinRect.right-glWinRect.left)/(ogl_xmax-ogl_xmin);
	*y=-(gly-ogl_ymax)*(glWinRect.bottom-glWinRect.top)/(ogl_ymax-ogl_ymin);
}

void glWin::win2gl(int winx, int winy, float *glx, float *gly)
{
	*glx = winx*(ogl_xmax-ogl_xmin)/(glWinRect.right-glWinRect.left) + ogl_xmin;
	*gly = -winy*(ogl_ymax-ogl_ymin)/(glWinRect.bottom-glWinRect.top) + ogl_ymax;
}

int glWin::OnCreate(LPCREATESTRUCT lpCreateStruct) 
{
	if (CWnd::OnCreate(lpCreateStruct) == -1)
		return -1;
	
	// TODO: Add your specialized creation code here
	return 0;
}

void glWin::OnMouseMove(UINT nFlags, CPoint point)
{
	// TODO: Add your message handler code here and/or call default
	CPoint scrPoint = point;
	ClientToScreen(&scrPoint);
	//CursorPos = scrPoint;
	//TRACE("glWin Mousemove %d %d\n",scrPoint.x,scrPoint.y);

	CWnd::OnMouseMove(nFlags, point);
}

```

Annotation parts can be removed which is not a necessary part of a glWin.















