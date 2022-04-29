/*  $Header: /nfs/kodak/kc1/pjs/src/X11/catclock/motif/RCS/catclock.c,v 1.7 91/11/26 11:46:54 pjs Exp Locker: pjs $ */

/*  Copyright 1985 Massachusetts Institute of Technology */

  

/*

 *  catclock.c MIT Project Athena, X Window system clock.

 *

 *  This program provides the user with a small

 *  window contining a digital clock with day and date.

 *  Parameters are variable from the command line.

 *

 *   Author:    Tony Della Fera, DEC

 *        September, 1984

 *   Hacked up by a cast of thousands....

 *

 *   Ported to X11 & Motif :

 *    Philip J. Schneider, DEC

 *      1990

 */

#include <stdio.h>

#include <stdlib.h>

#include <math.h>

#include <pwd.h>

#include <unistd.h>

  

#include <gtk/gtk.h>

  

/*

 *  X11 includes

 */

#include <X11/Xlib.h>

#include <X11/Xos.h>

#include <X11/Xutil.h>

#include <X11/cursorfont.h>

#include <X11/Intrinsic.h>

#include <X11/Shell.h>

#include <X11/StringDefs.h>

  

/*

 *  Motif includes

 */

#include <Xm/Xm.h>

#include <Xm/Form.h>

#include <Xm/DrawingA.h>

#include <Xm/PushB.h>

#include <Xm/ToggleB.h>

#include <Xm/MenuShell.h>

#include <Xm/RowColumn.h>

#include <Xm/Separator.h>

  

#include "alarm.h"

  

/*

 *  Cat bitmap includes

 */

#include "catback.xbm"

#include "catwhite.xbm"

#include "cattie.xbm"

#include "eyes.xbm"    

#include "tail.xbm"

  

/*

 *  Icon pixmap includes

 */

#include "cat.xbm"

  

/*

 *  Patch info

 */

#include "patchlevel.h"

  

/*

 *  Tempo tracker dependencies

 */

#if WITH_TEMPO_TRACKER

#include <sys/time.h>

#include <pthread.h>

#include <pulse/simple.h>

#include <pulse/error.h>

#include <aubio/aubio.h>

#define BUFSIZE 256

#define HOPSIZE 256

#define SAMPLERATE 44100

#endif

  

/*

 *  Cat body part pixmaps

 */

static Pixmap   *eyePixmap  = (Pixmap *)NULL;     /*  Array of eyes     */

static Pixmap   *tailPixmap = (Pixmap *)NULL;     /*  Array of tails    */

  

/*

 *  Cat GC's

 */

static GC       catGC;                    /*  For drawing cat's body    */

static GC       tailGC;                   /*  For drawing cat's tail    */

static GC       eyeGC;                    /*  For drawing cat's eyes    */

  

/*

 *  Default cat dimension stuff -- don't change sizes!!!!

 */

#define DEF_N_TAILS         16            /*  Default resolution        */

#define TAIL_HEIGHT         89            /*  Tail pixmap height        */

#define DEF_CAT_WIDTH       150           /*  Cat body pixmap width     */

#define DEF_CAT_HEIGHT      300           /*  Cat body pixmap height    */

#define DEF_CAT_BOTTOM      210           /*  Distance to cat's butt    */

  

/*

 *  Analog clock width and height

 */

#define DEF_ANALOG_WIDTH    164           /*  Chosen for historical     */

#define DEF_ANALOG_HEIGHT   164           /*  reasons :-)               */

  

/*

 *  Clock hand stuff

 */

#define VERTICES_IN_HANDS    4             /*  Hands are triangles      */

#define SECOND_HAND_FRACT    90            /*  Percentages of radius    */

#define MINUTE_HAND_FRACT    70    

#define HOUR_HAND_FRACT      40

#define HAND_WIDTH_FRACT     7

#define SECOND_WIDTH_FRACT   5

  

#define SECOND_HAND_TIME     30         /*  Update limit for second hand*/

  

static int     centerX = 0;             /*  Window coord origin of      */

static int     centerY = 0;             /*  clock hands.                */

  

static int     radius;                  /*  Radius of clock face        */

  

static int     secondHandLength;        /*  Current lengths and widths  */

static int     minuteHandLength;

static int     hourHandLength;

static int     handWidth;    

static int     secondHandWidth;

  

#define SEG_BUFF_SIZE        128                /*  Max buffer size     */

static int      numSegs = 0;                    /*  Segments in buffer  */

static XPoint   segBuf[SEG_BUFF_SIZE];          /*  Buffer              */

static XPoint   *segBufPtr = (XPoint *)NULL;    /*  Current pointer     */    

  

/*

 *  Padding defaults

 */

#define DEF_ANALOG_PADDING      8       /*  Radius padding for analog   */

  

/*

 *  Default font for digital display

 */

#define DEF_DIGITAL_FONT    "fixed"    

  

/*

 *  Alarm stuff

 */

#define DEF_ALARM_FILE        "/.Catclock"        /*  Alarm settings file */

#define DEF_ALARM_PERIOD    60        

  

static Boolean  alarmOn    = False;     /*  Alarm set?                  */

static Boolean  alarmState = False;     /*  Seems to be unused          */

static char     alarmBuf[BUFSIZ];       /*  Path name to alarm file     */

  

/*

 *  Time stuff

 */

static struct tm tm;                    /*  What time is it?            */

static struct tm otm;                   /*  What time was it?           */

  

/*

 *  X11 Stuff

 */

static Window        clockWindow = (Window)NULL;

static XtAppContext  appContext;

       Display       *dpy;

       Window        root;

       int           screen;

  

       GC       gc;                     /*  For tick-marks, text, etc.  */

static GC       handGC;                 /*  For drawing hands           */

static GC       eraseGC;                /*  For erasing hands           */

static GC       highGC;                 /*  For hand borders            */

  
  

#define UNINIT            -1

static int      winWidth  = UNINIT;     /*  Global window width         */

static int      winHeight = UNINIT;     /*  Global window height        */

  
  

/*

 *  Resources user can set in addition to normal Xt resources

 */

typedef struct {

    XFontStruct         *font;                  /*  For alarm & analog  */

  

    Pixel               foreground;             /*  Foreground          */

    Pixel               background;             /*  Background          */

  

    Pixel               highlightColor;         /*  Hand outline/border */

    Pixel               handColor;              /*  Hand interior       */

    Pixel               catColor;               /*  Cat's body          */

    Pixel               detailColor;            /*  Cat's paws, belly,  */

                                                /*  face, and eyes.     */

    Pixel               tieColor;               /*  Cat's tie color     */

    int                 nTails;                 /*  Tail/eye resolution */

  

    int                 padding;                /*  Font spacing        */

  

    int                 update;                 /*  Seconds between     */

                                                /*  updates             */

    Boolean             alarmSet;               /*  Alarm on?           */

    Boolean             alarmBell;              /*  Audible alarm?      */

    char                *alarmFile;             /*  Alarm setting file  */

    int                 alarmBellPeriod;        /*  How long to ring    */

                                                /*  alarm               */

    Boolean             chime;                  /*  Chime on hour?      */

  

    int                 help;                   /*  Display syntax      */

} ApplicationData, *ApplicationDataPtr;

  

static ApplicationData appData;

  
  
  

/*

 *  Miscellaneous stuff

 */

#define TWOPI                   (2.0 * M_PI)            /*  2.0 * M_PI  */

  

#define DEF_UPDATE              60

  

#define min(a, b)               ((a) < (b) ? (a) : (b))

#define max(a, b)               ((a) > (b) ? (a) : (b))

  

static Boolean  noSeconds      = False;    /*  Update time >= 1 minute? */

static Boolean  evenUpdate     = False;    /*  Even update interval?    */

static Boolean  showSecondHand = False;    /*  Display second hand?     */

  

static Boolean  iconified      = False;    /*  Clock iconified?         */

  
  

#if WITH_TEMPO_TRACKER

static float phase = 0.5;

static float bpm = 120.0;

static float correction = 0.0;

Boolean last_time_initialized = True;

struct timeval last_time;

static int direction = 1;

#endif

  

static void ParseGeometry(Widget, int, int, int);

static int Round(double);

static void Syntax(char *);

static void InitializeCat(Pixel, Pixel, Pixel);

static GC CreateTailGC(void);

static GC CreateEyeGC(void);

static Pixmap CreateTailPixmap(double);

static Pixmap CreateEyePixmap(double);

static void EraseHands(Widget, struct tm *);

static void HandleExpose(Widget, XtPointer, XtPointer);

static void HandleInput(Widget, XtPointer, XtPointer);

static void Tick(Widget, int);

static void AlarmSetCallback(Widget, XtPointer, XtPointer);

static void AlarmBellCallback(Widget, XtPointer, XtPointer);

static void ChimeCallback(Widget, XtPointer, XtPointer);

static void AckAlarmCallback(Widget, XtPointer, XtPointer);

static void RereadAlarmCallback(Widget, XtPointer, XtPointer);

static void EditAlarmCallback(Widget, XtPointer, XtPointer);

static void ExitCallback(Widget, XtPointer, XtPointer);

static void MapCallback(Widget, XtPointer, XEvent *, Boolean *);

static void SetSeg(int, int, int, int);

#if WITH_TEMPO_TRACKER

static void *TempoTrackerThread();

#endif

  

static void activate (GtkApplication *app, gpointer user_data)

{

  GtkWidget *window;

  

  window = gtk_application_window_new (app);

  gtk_window_set_title (GTK_WINDOW (window), "Neo Catclock");

  

  // Need to use gtk_widget_set_size_request here instead of set_default_size.

  // Otherwise it interfers with gtk_window_set_resizable.

  gtk_widget_set_size_request(window, DEF_CAT_WIDTH, DEF_CAT_HEIGHT);

  gtk_window_set_resizable (GTK_WINDOW(window), FALSE);

  

  // Connect Motif Event Loop so original app displays properly.

  g_idle_add (XtAppMainLoop, appContext);

  

  gtk_window_present (GTK_WINDOW (window));

}

  

int main(argc, argv)

    int         argc;

    char        *argv[];

{

    int         n;

    Arg         args[10];

    Widget      topLevel, form, canvas;

    int         stringWidth = 0,

                stringAscent = 0, stringDescent = 0;

    XGCValues   gcv;

    u_long      valueMask;

    XmFontList  fontList = (XmFontList)NULL;

  

    static XtResource    resources[] = {

        { XtNfont, XtCFont, XtRFontStruct, sizeof(XFontStruct *),

          XtOffset(ApplicationDataPtr, font),

          XtRString, (XtPointer)DEF_DIGITAL_FONT },

  

        { XtNforeground, XtCForeground, XtRPixel, sizeof(Pixel),

          XtOffset(ApplicationDataPtr, foreground),

          XtRString, (XtPointer)"XtdefaultForeground" },

  

        { XtNbackground, XtCBackground, XtRPixel, sizeof(Pixel),

          XtOffset(ApplicationDataPtr, background),

          XtRString, (XtPointer)"XtdefaultBackground" },

  

        { "highlight", "HighlightColor", XtRPixel, sizeof(Pixel),

          XtOffset(ApplicationDataPtr, highlightColor),

          XtRString, (XtPointer)"XtdefaultForeground" },

        { "hands", "Hands", XtRPixel, sizeof(Pixel),

          XtOffset(ApplicationDataPtr, handColor),

          XtRString, (XtPointer)"XtdefaultForeground" },

        { "catColor", "CatColor", XtRPixel, sizeof(Pixel),

          XtOffset(ApplicationDataPtr, catColor),

          XtRString, (XtPointer)"XtdefaultForeground" },

        { "detailColor", "DetailColor", XtRPixel, sizeof(Pixel),

          XtOffset(ApplicationDataPtr, detailColor),

          XtRString, (XtPointer)"XtdefaultBackground" },

        { "tieColor", "TieColor", XtRPixel, sizeof(Pixel),

          XtOffset(ApplicationDataPtr, tieColor),

          XtRString, (XtPointer)"XtdefaultBackground" },

  

        { "padding", "Padding", XtRInt, sizeof(int),

          XtOffset(ApplicationDataPtr, padding),

          XtRImmediate, (XtPointer)UNINIT },

        { "nTails", "NTails", XtRInt, sizeof(int),

          XtOffset(ApplicationDataPtr, nTails),

          XtRImmediate, (XtPointer)DEF_N_TAILS },

        { "update", "Update", XtRInt, sizeof(int),

          XtOffset(ApplicationDataPtr, update),

          XtRImmediate, (XtPointer)DEF_UPDATE },

        { "alarm", "Alarm", XtRBoolean, sizeof(Boolean),

          XtOffset(ApplicationDataPtr, alarmSet),

          XtRImmediate, (XtPointer)False },

        { "bell", "Bell", XtRBoolean, sizeof(Boolean),

          XtOffset(ApplicationDataPtr, alarmBell),

          XtRImmediate, (XtPointer)False },

        { "file", "File", XtRString, sizeof(char *),

          XtOffset(ApplicationDataPtr, alarmFile),

          XtRString, (XtPointer)NULL },

        { "period", "Period", XtRInt, sizeof(int),

          XtOffset(ApplicationDataPtr, alarmBellPeriod),

          XtRImmediate, (XtPointer)DEF_ALARM_PERIOD },

        { "chime", "Chime", XtRBoolean, sizeof(Boolean),

          XtOffset(ApplicationDataPtr, chime),

          XtRImmediate, (XtPointer)False },

  

        { "help", "Help", XtRBoolean, sizeof(Boolean),

          XtOffset(ApplicationDataPtr, help),

          XtRImmediate, (XtPointer)False},

    };

    static XrmOptionDescRec options[] = {

        { "-bg",          "*background",   XrmoptionSepArg, (XtPointer)NULL },

        { "-background",  "*background",   XrmoptionSepArg, (XtPointer)NULL },

  

        { "-fg",          "*foreground",   XrmoptionSepArg, (XtPointer)NULL },

        { "-foreground",  "*foreground",   XrmoptionSepArg, (XtPointer)NULL },

  

        { "-fn",          "*font",         XrmoptionSepArg, (XtPointer)NULL },

        { "-font",        "*font",         XrmoptionSepArg, (XtPointer)NULL },

  

        { "-hl",          "*highlight",    XrmoptionSepArg, (XtPointer)NULL },

        { "-highlight",   "*highlight",    XrmoptionSepArg, (XtPointer)NULL },

  

        { "-hd",          "*hands",        XrmoptionSepArg, (XtPointer)NULL },

        { "-hands",       "*hands",        XrmoptionSepArg, (XtPointer)NULL },

  

        { "-catcolor",    "*catColor",     XrmoptionSepArg, (XtPointer)NULL },

        { "-detailcolor", "*detailColor",  XrmoptionSepArg, (XtPointer)NULL },

        { "-tiecolor",    "*tieColor",     XrmoptionSepArg, (XtPointer)NULL },

  

        { "-padding",     "*padding",      XrmoptionSepArg, (XtPointer)NULL },

        { "-ntails",      "*nTails",       XrmoptionSepArg, (XtPointer)NULL },

        { "-update",      "*update",       XrmoptionSepArg, (XtPointer)NULL },

        { "-alarm",       "*alarm",        XrmoptionNoArg,  (XtPointer)"on" },

        { "-bell",        "*bell",         XrmoptionNoArg,  (XtPointer)"on" },

        { "-file",        "*file",         XrmoptionSepArg, (XtPointer)NULL },

        { "-period",      "*period",       XrmoptionSepArg, (XtPointer)NULL },

        { "-chime",       "*chime",        XrmoptionNoArg,  (XtPointer)"on" },

        { "-help",        "*help",         XrmoptionNoArg,  (XtPointer)"on" },

    };

    /*

     *  Hack to make Saber-C work with Xt correctly

     */

    argv[0] = "catclock";

  

    /*

     *  Open up the system

     */

    topLevel = XtAppInitialize(&appContext, "Catclock",

                               (XrmOptionDescList)(&options[0]),

                               XtNumber(options),

                               &argc, argv, NULL,

                               NULL, 0);

    /*

     *  Get resources . . .

     */

    XtGetApplicationResources(topLevel, &appData, resources,

                              XtNumber(resources), NULL, 0);

  

    /*

     *  Print help message and exit if asked

     */

    if (appData.help) {

        Syntax(argv[0]);

    }

  

    /*

     *  Save away often-used X stuff

     */

    dpy    = XtDisplay(topLevel);

    screen = DefaultScreen(dpy);

    root   = DefaultRootWindow(dpy);

  

    /*

     *  See if user specified iconic startup for clock

     */

    n = 0;

    XtSetArg(args[n], XmNiconic, &iconified);    n++;

    XtGetValues(topLevel, args, n);

  

    /*

     *  Create icon pixmap

     */

    {

        Pixmap   iconPixmap;

        char    *data;

        u_int    width = 0, height = 0;

  

        data   = cat_bits;

        width  = cat_width;

        height = cat_height;

  

        iconPixmap = XCreateBitmapFromData(dpy, root,

                                           data, width, height);

        n = 0;

        XtSetArg(args[n], XmNiconPixmap, iconPixmap);    n++;

        XtSetValues(topLevel, args, n);

    }

    /*

     *  Get the font info

     */

    if (appData.font == (XFontStruct *)NULL) {

        appData.font = XLoadQueryFont(dpy, DEF_DIGITAL_FONT);

        if (appData.font == (XFontStruct *)NULL) {

            fprintf(stderr,

                    "catclock : Unable to load default font %s. Please re-run with another font\n",

                    DEF_DIGITAL_FONT);

            exit(-1);

        }

    }

    fontList = XmFontListCreate(appData.font, XmSTRING_ISO8859_1);

    if (fontList == (XmFontList)NULL) {

        fprintf(stderr, "catclock : Unable to create font list -- exiting\n");

        exit(-1);

    }

  

    /*

     *  Padding is space between clock face and

     *  edge of cat's belly.

     */

    if (appData.padding == UNINIT) {

        appData.padding = DEF_ANALOG_PADDING;

    }

    /*

     *  Bound the number of tails

     */

    if (appData.nTails < 1) {

        appData.nTails = 1;

    }

    if (appData.nTails > 60) {

        appData.nTails = 60;

    }

    /*

     *  Update rate depends on number of tails,

     *  so tail swings at approximately 60 hz.

     */

    appData.update = (int)(0.5 + 1000.0 / appData.nTails);

    /*

     *  Drawing the second hand on the cat is ugly.

     */

    showSecondHand = False;

    /*

     *  "ParseGeometry"  looks at the user-specified geometry

     *  specification string, and attempts to apply it in a rational

     *  fashion to the clock in whatever mode it happens to be in.

     *  For example, for analog mode, any sort of geometry is OK, but

     *  the cat must be a certain size, although the x and y origins

     *  should not be ignored, etc.

     */

    ParseGeometry(topLevel, stringWidth, stringAscent, stringDescent);

  

    /*

     *  Create widgets for catclock :

     *

     *

     *  Outermost widget is a form widget.

     */

    n = 0;

    form = XmCreateForm(topLevel, "form", args, n);

    XtManageChild(form);

  

    /*

     *  "canvas" is the display widget

     */

    n = 0;

    XtSetArg(args[n], XmNtopAttachment,    XmATTACH_FORM);    n++;

    XtSetArg(args[n], XmNbottomAttachment, XmATTACH_FORM);    n++;

    XtSetArg(args[n], XmNleftAttachment,   XmATTACH_FORM);    n++;

    XtSetArg(args[n], XmNrightAttachment,  XmATTACH_FORM);    n++;

    XtSetArg(args[n], XmNforeground,       appData.foreground);    n++;

    XtSetArg(args[n], XmNbackground,       appData.background);    n++;

    canvas = XmCreateDrawingArea(form, "drawingArea", args, n);

    XtManageChild(canvas);

  

    /*

     *  Make all the windows, etc.

     */

    XtRealizeWidget(topLevel);

  

    /*

     *  When clock is first mapped, we can start

     *  the ticking.  This handler is removed following

     *  that first map event.  This approach is

     *  necessary (I think) to make the clock

     *  start up correctly with different window

     *  managers, or without one, for that matter.

     */

    XtAddEventHandler(topLevel, StructureNotifyMask,

                      False, MapCallback, (XtPointer)NULL);

  

    /*

     *  Cache the window associated with the XmDrawingArea

     */

    clockWindow = XtWindow(canvas);

  

    /*

     *  Supply a cursor for this application

     */

    {

        Cursor arrow = XCreateFontCursor(dpy, XC_top_left_arrow);

  

        XDefineCursor(dpy, clockWindow, arrow);

    }

  

    /*

     *  Check if update interval is even or odd # of seconds

     */

    if (appData.update > 1 &&

        ((60 / appData.update) * appData.update == 60)) {

        evenUpdate = True;

    }

  

    /*

     *  Set the sizes of the hands

     */

  

    winWidth  = DEF_CAT_WIDTH;

    winHeight = DEF_CAT_HEIGHT;

    radius = Round((min(winWidth,

                        winHeight)-(2 * appData.padding)) / 3.45);

    secondHandLength =  ((SECOND_HAND_FRACT  * radius)  / 100);

    minuteHandLength =  ((MINUTE_HAND_FRACT  * radius)  / 100);

    hourHandLength   =  ((HOUR_HAND_FRACT    * radius)  / 100);

    handWidth        =  ((HAND_WIDTH_FRACT   * radius) / 100) * 2;

    secondHandWidth  =  ((SECOND_WIDTH_FRACT * radius) / 100);

    centerX = winWidth  / 2;

    centerY = winHeight / 2;

  

    /*

     *  Create the GC's

     */

    valueMask = GCForeground | GCBackground | GCFont |

                GCLineWidth  | GCGraphicsExposures;

    gcv.background = appData.background;

    gcv.foreground = appData.foreground;  

    gcv.graphics_exposures = False;

    gcv.line_width = 0;

    if (appData.font != NULL) {

        gcv.font = appData.font->fid;

    } else {

        valueMask &= ~GCFont;    /* use server default font */

    }

    gc = XCreateGC(dpy, clockWindow,  valueMask, &gcv);

    valueMask = GCForeground | GCLineWidth ;

    gcv.foreground = appData.background;

    eraseGC = XCreateGC(dpy, clockWindow,  valueMask, &gcv);

    gcv.foreground = appData.highlightColor;

    highGC = XCreateGC(dpy, clockWindow,  valueMask, &gcv);

    valueMask = GCForeground;

    gcv.foreground = appData.handColor;

    handGC = XCreateGC(dpy, clockWindow,  valueMask, &gcv);

  

    /*

     *  Make sure we have an alarm file.  If not

     *  specified in command line or .Xdefaults file, then

     *  use default of "$HOME/DEF_ALARM_FILE"

     */

    if (!appData.alarmFile) {

        char          *cp;

        struct passwd *pw;

  

        if ((cp = getenv("HOME"))) {

            strcpy(alarmBuf, cp);

        } else if ((pw = getpwuid(getuid()))) {

            strcpy(alarmBuf, pw->pw_dir);

        } else {

            *alarmBuf = 0;

        }

  

        strcat(appData.alarmFile = alarmBuf, DEF_ALARM_FILE);

    }

  

    /*

     *  Set up the alarm and alarm bell

     */

    InitBellAlarm(clockWindow,

                  winWidth, winHeight,

                  appData.font, fontList,

                  appData.foreground, appData.background,

                  &alarmState, &alarmOn);

    SetAlarm(appData.alarmSet ? appData.alarmFile : NULL);

    SetBell(appData.alarmBell ? appData.alarmBellPeriod : 0);

    /*

     *  Create cat pixmaps, etc.

     */

  

    InitializeCat(appData.catColor,

                  appData.detailColor,

                  appData.tieColor);

  

    /*

     *  Finally, install necessary callbacks

     */

    {

        XtAddCallback(canvas, XmNexposeCallback, HandleExpose, NULL);

        XtAddCallback(canvas, XmNinputCallback,  HandleInput,  NULL);

    }

  

#if WITH_TEMPO_TRACKER

    pthread_t thread;

    void *arg;

    pthread_create(&thread, NULL, TempoTrackerThread, arg);

#endif

  

    /*

     *  Start processing events

     */

    GtkApplication *app;

    int status;

    app = gtk_application_new ("org.happy-ferret.catclock", G_APPLICATION_FLAGS_NONE);

    g_signal_connect (app, "activate", G_CALLBACK (activate), NULL);

    status = g_application_run (G_APPLICATION (app), argc, argv);

    g_object_unref (app);

  

    return status;

}

  

#if WITH_TEMPO_TRACKER

static void *TempoTrackerThread() {

  static const pa_sample_spec ss = {

    .format = PA_SAMPLE_FLOAT32,

    .rate = SAMPLERATE,

    .channels = 1

  };

  pa_simple *s = NULL;

  int error;

  if (!(s = pa_simple_new(NULL, "Cat Clock", PA_STREAM_RECORD, NULL, "record", &ss, NULL, NULL, &error))) {

    fprintf(stderr, __FILE__": pa_simple_new() failed: %s\n", pa_strerror(error));

    pthread_exit(NULL);

  }

  

  aubio_tempo_t *tempo = new_aubio_tempo("default", BUFSIZE, HOPSIZE, SAMPLERATE);

  fvec_t *tempo_out = new_fvec(2);

  for (;;) {

    float buf[BUFSIZE];

    pa_simple_read(s, buf, sizeof(buf), &error);

    fvec_t *aubio_buf = new_fvec(BUFSIZE);

    aubio_buf->data = buf;

    aubio_tempo_do(tempo, aubio_buf, tempo_out);

    float confidence = aubio_tempo_get_confidence(tempo);

    Boolean is_beat = tempo_out->data[0] != 0.0;

    if (confidence > 0.1) {

      bpm = aubio_tempo_get_bpm(tempo);

    }

  

    // Time delta

    struct timeval now;

    gettimeofday(&now, NULL);

    if (!last_time_initialized) {

      last_time = now;

      last_time_initialized = True;

    }

    struct timeval time_delta;

    timersub(&now, &last_time, &time_delta);

    last_time = now;

  

    // Speed correction

    //

    // We want beat to hit at 0.0, 0.5 or "1.0"

    // so we trying to detect if it was too late this time

    // and attenuate the speed to make animation sync

    // with the beat better when it occur next time.

  

    if (confidence > 0.1) {

      if (is_beat) {

        float phase_delta = 0;

        if (phase < 0.25) {

          phase_delta = phase - 0.0;

        } else if (phase > 0.25 && phase < 0.75) {

          phase_delta = phase - 0.5;

        } else if (phase > 0.75) {

          phase_delta = phase - 1.0;

        }

        correction = phase_delta * -direction;

      }

    }

  

    float time_delta_s = ((float)time_delta.tv_usec * 0.000001);

  

    float correction_amplification = 20.0;

  

    float speed = bpm / 60.0 * 0.5 + (correction * correction_amplification * time_delta_s);

  

    // Move phase

  

    phase += time_delta_s * speed * direction;

    if (direction == 1 && phase >= 1.0) {

      phase = 1.0;

      direction = -1;

    } else if (direction == -1 && phase < 0.0) {

      phase = 0.0;

      direction = 1;

    }

  }

  pthread_exit(NULL);

}

#endif

  

static void ParseGeometry(topLevel, stringWidth, stringAscent, stringDescent)

    Widget      topLevel;

    int         stringWidth, stringAscent, stringDescent;

{

    int         n;

    Arg         args[10];

    char        *geomString = NULL;

    char        *geometry,

                xString[80], yString[80],

                widthString[80], heightString[80];

  

    /*

     *  Grab the geometry string from the topLevel widget

     */

    n = 0;

    XtSetArg(args[n], XmNgeometry, &geomString);    n++;

    XtGetValues(topLevel, args, n);

  

    /*

     * Static analysis tags this as a memory leak...

     * Must be dynamically allocated; otherwise, the geom string

     * would be out of scope by the time XtRealizeWidget is called,

     * and you would get a garbage geometry specification, at best.

     */

    geometry = malloc(80);

  

    if (geomString == NULL) {

        /*

         *  User didn't specify any geometry, so we

         *  use the default.

         */

        sprintf(geometry, "%dx%d", DEF_CAT_WIDTH, DEF_CAT_HEIGHT);

        }

  

    /*

     *  Stash the width and height in some globals (ugh!)

     */

    sscanf(geometry, "%dx%d", &winWidth, &winHeight);

    /*

     *  Set the geometry of the topLevel widget

     */

    n = 0;

    XtSetArg(args[n], XmNwidth,     DEF_CAT_WIDTH);    n++;

    XtSetArg(args[n], XmNheight,    DEF_CAT_HEIGHT);   n++;

    XtSetArg(args[n], XmNminWidth,  DEF_CAT_WIDTH);    n++;

    XtSetArg(args[n], XmNminHeight, DEF_CAT_HEIGHT);   n++;

    XtSetArg(args[n], XmNmaxWidth,  DEF_CAT_WIDTH);    n++;

    XtSetArg(args[n], XmNmaxHeight, DEF_CAT_HEIGHT);   n++;

    XtSetArg(args[n], XmNgeometry,  geometry);         n++;

    XtSetValues(topLevel, args, n);

  

}

  

/*

 *  DrawLine - Draws a line.

 *

 *  blankLength is the distance from the center which the line begins.

 *  length is the maximum length of the hand.

 *  fractionOfACircle is a fraction between 0 and 1 (inclusive) indicating

 *  how far around the circle (clockwise) from high noon.

 *

 *  The blankLength feature is because I wanted to draw tick-marks around the

 *  circle (for seconds).  The obvious means of drawing lines from the center

 *  to the perimeter, then erasing all but the outside most pixels doesn't

 *  work because of round-off error (sigh).

 */

static void DrawLine(blankLength, length, fractionOfACircle)

    int         blankLength;

    int         length;

    double      fractionOfACircle;

{

    double      angle, cosAngle, sinAngle;

  

    /*

     *  A full circle is 2 M_PI radians.

     *  Angles are measured from 12 o'clock, clockwise increasing.

     *  Since in X, +x is to the right and +y is downward:

     *

     *    x = x0 + r * sin(theta)

     *    y = y0 - r * cos(theta)

     *

     */

    angle = TWOPI * fractionOfACircle;

    cosAngle = cos(angle);

    sinAngle = sin(angle);

    SetSeg(centerX + (int)(blankLength * sinAngle),

           centerY - (int)(blankLength * cosAngle),

           centerX + (int)(length * sinAngle),

           centerY - (int)(length * cosAngle));

}

  

/*

 *  DrawHand - Draws a hand.

 *

 *  length is the maximum length of the hand.

 *  width is the half-width of the hand.

 *  fractionOfACircle is a fraction between 0 and 1 (inclusive) indicating

 *  how far around the circle (clockwise) from high noon.

 *

 */

static void DrawHand(length, width, fractionOfACircle)

    int         length, width;

    double      fractionOfACircle;

{

    double      angle, cosAngle, sinAngle;

    double      ws, wc;

    int         x, y, x1, y1, x2, y2;

    /*

     *  A full circle is 2 PI radians.

     *  Angles are measured from 12 o'clock, clockwise increasing.

     *  Since in X, +x is to the right and +y is downward:

     *

     *    x = x0 + r * sin(theta)

     *    y = y0 - r * cos(theta)

     *

     */

    angle = TWOPI * fractionOfACircle;

    cosAngle = cos(angle);

    sinAngle = sin(angle);

  

    /*

     * Order of points when drawing the hand.

     *

     *        1,4

     *        / \

     *       /   \

     *      /     \

     *    2 ------- 3

     */

    wc = width * cosAngle;

    ws = width * sinAngle;

    SetSeg(x = centerX + Round(length * sinAngle),

           y = centerY - Round(length * cosAngle),

           x1 = centerX - Round(ws + wc),

           y1 = centerY + Round(wc - ws));  /* 1 ---- 2 */

    /* 2 */

    SetSeg(x1, y1,

           x2 = centerX - Round(ws - wc),

           y2 = centerY + Round(wc + ws));  /* 2 ----- 3 */

    SetSeg(x2, y2, x, y);    /* 3 ----- 1(4) */

}

  

/*

 *  DrawSecond - Draws the second hand (diamond).

 *

 *  length is the maximum length of the hand.

 *  width is the half-width of the hand.

 *  offset is direct distance from Center to tail end.

 *  fractionOfACircle is a fraction between 0 and 1 (inclusive) indicating

 *  how far around the circle (clockwise) from high noon.

 *

 */

static void DrawSecond(length, width, offset, fractionOfACircle)

    int         length, width, offset;

    double      fractionOfACircle;

{

    double      angle, cosAngle, sinAngle;

    double      ms, mc, ws, wc;

    int         mid;

    int         x, y;

    /*

     *  A full circle is 2 PI radians.

     *  Angles are measured from 12 o'clock, clockwise increasing.

     *  Since in X, +x is to the right and +y is downward:

     *

     *    x = x0 + r * sin(theta)

     *    y = y0 - r * cos(theta)

     *

     */

    angle = TWOPI * fractionOfACircle;

    cosAngle = cos(angle);

    sinAngle = sin(angle);

  

    /*

     * Order of points when drawing the hand.

     *

     *        1,5

     *        / \

     *       /   \

     *      /     \

     *    2<       >4

     *      \     /

     *       \   /

     *        \ /

     *    -    3

     *    |

     *    |

     *   offset

     *    |

     *    |

     *    -     + center

     */

    mid = (length + offset) / 2;

    mc = mid * cosAngle;

    ms = mid * sinAngle;

    wc = width * cosAngle;

    ws = width * sinAngle;

    /*1 ---- 2 */

    SetSeg(x = centerX + Round(length * sinAngle),

           y = centerY - Round(length * cosAngle),

           centerX + Round(ms - wc),

           centerY - Round(mc + ws) );

    SetSeg(centerX + Round(ms - wc),

           centerY - Round(mc + ws),

           centerX + Round(offset * sinAngle),

           centerY - Round(offset * cosAngle)); /* 2-----3 */

  

    SetSeg(centerX + Round(offset *sinAngle),

           centerY - Round(offset * cosAngle), /* 3-----4 */

           centerX + Round(ms + wc),

           centerY - Round(mc - ws));      

    segBufPtr->x = x;

    segBufPtr++->y = y;

    numSegs++;

}

  

static void SetSeg(x1, y1, x2, y2)

    int x1, y1, x2, y2;

{

    segBufPtr->x   = x1;

    segBufPtr++->y = y1;

    segBufPtr->x   = x2;

    segBufPtr++->y = y2;

  

    numSegs += 2;

}

  

/*

 *  Draw the clock face (every fifth tick-mark is longer

 *  than the others).

 */

static void DrawClockFace(secondHand, radius)

    int         secondHand;

    int         radius;

{

    int         i;

    int         delta = (radius - secondHand) / 3;

  

    segBufPtr = segBuf;

    numSegs = 0;

  

    XFillRectangle(dpy, clockWindow, catGC,

                   0, 0, winWidth, winHeight);

    segBufPtr = segBuf;

    numSegs = 0;

    DrawBell(False);

}

  

static int Round(x)

    double x;

{

    return (x >= 0.0 ? (int)(x + 0.5) : (int)(x - 0.5));

}

  

/*

 * Report the syntax for calling catclock.

 */

static void Syntax(call)

    char *call;

{

    printf("Usage: %s [toolkitoptions]\n", call);

    printf("       [-alarm]  [-bell]  [-chime]\n");

    printf("       [-file <alarm file>]  [-period <seconds>]\n");

    printf("       [-hl <color>]  [-hd <color>]\n");

    printf("       [-catcolor <color>]\n");

    printf("       [-detailcolor <color>]\n");

    printf("       [-tiecolor <color>]\n");

    printf("       [-padding <pixels>]\n");

    printf("       [-update <seconds>]\n");

    printf("       [-ntails <number>]\n");

    printf("       [-help]\n\n");

  

    exit(0);

}

  
  

static void InitializeCat(catColor, detailColor, tieColor)

    Pixel    catColor, detailColor, tieColor;

{

    Pixmap           catPix;

    Pixmap           catBack;

    Pixmap           catBackG;

    Pixmap           catWhite;

    Pixmap           catTie;

    int              fillStyle;

    int              i;

    XGCValues        gcv;

    unsigned long    valueMask;

    GC               gc1, gc2;

  

    catPix = XCreatePixmap(dpy, root, winWidth, winHeight,

                           DefaultDepth(dpy, screen));

    valueMask = GCForeground | GCBackground | GCGraphicsExposures;

  

    gcv.background = appData.background;

    gcv.foreground = catColor;  

    gcv.graphics_exposures = False;

    gcv.line_width = 0;

    gc1 = XCreateGC(dpy, root, valueMask, &gcv);

  

    fillStyle = FillOpaqueStippled;

    XSetFillStyle(dpy, gc1, fillStyle);

  

    catBackG = cairo_image_surface_create_for_data(catback_bits,       /* Data                                  */

                                                   CAIRO_FORMAT_A1,    /* Image format => 1 bit alpha per pixel */

                                                   catback_width,      /* Width                                 */

                                                   catback_height,     /* Height                                */    

                                                   4);                 /* Stride                                */

  

    catBack = XCreateBitmapFromData(dpy, root, catback_bits,

                                    catback_width, catback_height);

  

    XSetStipple(dpy, gc1, catBack);

    XSetTSOrigin(dpy, gc1, 0, 0);

  

    XFillRectangle(dpy, catPix, gc1,

                   0, 0, winWidth, winHeight);

  

    fillStyle = FillStippled;

    XSetFillStyle (dpy, gc1, fillStyle);

    XFreeGC(dpy, gc1);

  

    valueMask = GCForeground | GCBackground | GCGraphicsExposures;

  

    gcv.background         = appData.background;

    gcv.foreground         = detailColor;  

    gcv.graphics_exposures = False;

    gcv.line_width         = 0;

    gc2 = XCreateGC(dpy, root, valueMask, &gcv);

    fillStyle = FillStippled;

    XSetFillStyle(dpy, gc2, fillStyle);

    catWhite = XCreateBitmapFromData(dpy, root, catwhite_bits,

                                     catwhite_width, catwhite_height);

    XSetStipple(dpy, gc2, catWhite);

    XSetTSOrigin(dpy, gc2, 0, 0);

    XFillRectangle(dpy, catPix, gc2,

                   0, 0, winWidth, winHeight);

    XFreeGC(dpy, gc2);

    gcv.background         = appData.background;

    gcv.foreground         = tieColor;  

    gcv.graphics_exposures = False;

    gcv.line_width         = 0;

    catGC = XCreateGC(dpy, root, valueMask, &gcv);

  

    fillStyle = FillStippled;

    XSetFillStyle (dpy, catGC, fillStyle);

    catTie = XCreateBitmapFromData(dpy, root, cattie_bits,

                                   cattie_width, cattie_height);

  

    XSetStipple(dpy, catGC, catTie);

    XSetTSOrigin(dpy, catGC, 0, 0);

    XFillRectangle(dpy, catPix, catGC, 0, 0, winWidth, winHeight);

    /*

     *  Now, let's create the Backround Pixmap for the Cat Clock using catGC

     *  We will use this pixmap to fill in the window backround.        

     */

    fillStyle = FillTiled;

    XSetFillStyle(dpy, catGC, fillStyle);

    XSetTile(dpy, catGC, catPix);

    XSetTSOrigin(dpy, catGC, 0, 0);

  

    /*

     *  Create the arrays of tail and eye pixmaps

     */

    tailGC = CreateTailGC();

    eyeGC  = CreateEyeGC();

  

    tailPixmap = (Pixmap *)malloc((appData.nTails + 1) * sizeof(Pixmap));

    eyePixmap  = (Pixmap *)malloc((appData.nTails + 1) * sizeof(Pixmap));

  

    for (i = 0; i <= appData.nTails; i++) {

  

        tailPixmap[i] = CreateTailPixmap(i * M_PI/(appData.nTails));

        eyePixmap[i]  = CreateEyePixmap(i * M_PI/(appData.nTails));

    }

}

  

static void UpdateEyesAndTail()

{

    static int  curTail = 0;    /*  Index into tail pixmap array       */

    static int  tailDir = 1;    /*  Left or right swing                */

  

    /*

     *  Draw new tail & eyes (Don't change values here!!)

     */

    XCopyPlane(dpy, tailPixmap[curTail], clockWindow,

               tailGC, 0, 0, winWidth, TAIL_HEIGHT,

//               tailGC, 0, 0, winWidth, tail_height,

               0, DEF_CAT_BOTTOM + 1, 0x1);

    XCopyPlane(dpy, eyePixmap[curTail], clockWindow,

               eyeGC, 0, 0, eyes_width, eyes_height,

               49, 30, 0x1);

  

    /*

     *  Figure out which tail & eyes are next

     */

#if WITH_TEMPO_TRACKER

    // Time delta

    struct timeval now;

    gettimeofday(&now, NULL);

    if (last_time_initialized) {

      struct timeval time_delta;

      timersub(&now, &last_time, &time_delta);

      if (time_delta.tv_usec > 30000) {

        // Seems like PulseAudio is sleeping

        // We will move phase ourself

        float speed = bpm / 60.0 * 0.5;

        phase += ((float)time_delta.tv_usec * 0.000001) * speed * direction;

        if (direction == 1 && phase >= 1.0) {

          phase = 1.0;

          direction = -1;

        } else if (direction == -1 && phase < 0.0) {

          phase = 0.0;

          direction = 1;

        }

        last_time = now;

      }

    }

    curTail = (int) (phase * appData.nTails);

#else

    if (curTail == 0 && tailDir == -1) {

        curTail = 1;

        tailDir = 1;

    } else if (curTail == appData.nTails && tailDir == 1) {

        curTail = appData.nTails - 1;

        tailDir = -1;

    } else {

        curTail += tailDir;

    }

#endif

}

  
  

static GC CreateTailGC()

{

    GC                 tailGC;

    XGCValues          tailGCV;

    unsigned long      valueMask;

  

    tailGCV.function           = GXcopy;

    tailGCV.plane_mask         = AllPlanes;

    tailGCV.foreground         = appData.catColor;

    tailGCV.background         = appData.background;

    tailGCV.line_width         = 15;

    tailGCV.line_style         = LineSolid;

    tailGCV.cap_style          = CapRound;

    tailGCV.join_style         = JoinRound;

    tailGCV.fill_style         = FillSolid;

    tailGCV.subwindow_mode     = ClipByChildren;

    tailGCV.clip_x_origin      = 0;

    tailGCV.clip_y_origin      = 0;

    tailGCV.clip_mask          = None;

    tailGCV.graphics_exposures = False;

  

    valueMask =

        GCFunction  | GCPlaneMask     | GCForeground  | GCBackground  |

        GCLineWidth | GCLineStyle     | GCCapStyle    | GCJoinStyle   |

        GCFillStyle | GCSubwindowMode | GCClipXOrigin | GCClipYOrigin |

        GCClipMask  | GCGraphicsExposures;

  

    tailGC = XCreateGC(dpy, clockWindow, valueMask, &tailGCV);

    return (tailGC);

}

  
  

static GC CreateEyeGC()

{

    GC                 eyeGC;

    XGCValues          eyeGCV;

    unsigned long      valueMask;

  

    eyeGCV.function           = GXcopy;

    eyeGCV.plane_mask         = AllPlanes;

    eyeGCV.foreground         = appData.catColor;

    eyeGCV.background         = appData.detailColor;

    eyeGCV.line_width         = 15;

    eyeGCV.line_style         = LineSolid;

    eyeGCV.cap_style          = CapRound;

    eyeGCV.join_style         = JoinRound;

    eyeGCV.fill_style         = FillSolid;

    eyeGCV.subwindow_mode     = ClipByChildren;

    eyeGCV.clip_x_origin      = 0;

    eyeGCV.clip_y_origin      = 0;

    eyeGCV.clip_mask          = None;

    eyeGCV.graphics_exposures = False;

  

    valueMask =

        GCFunction  | GCPlaneMask     | GCForeground  | GCBackground  |

        GCLineWidth | GCLineStyle     | GCCapStyle    | GCJoinStyle   |

        GCFillStyle | GCSubwindowMode | GCClipXOrigin | GCClipYOrigin |

        GCClipMask  | GCGraphicsExposures;

  

    eyeGC = XCreateGC(dpy, clockWindow, valueMask, &eyeGCV);

    return (eyeGC);

}

  
  

static Pixmap CreateTailPixmap(t)

    double      t;                      /*  "Time" parameter  */

{

    Pixmap      tailBitmap;

    GC          bitmapGC;

    double      sinTheta, cosTheta;     /*  Pendulum parameters */

    double      A = 0.4;

    double      omega = 1.0;

    double      phi = 3 * M_PI_2;

    double      angle;

  

    static XPoint tailOffset = { 74, -15 };

  

#define N_TAIL_PTS    7

    static XPoint  tail[N_TAIL_PTS] = { /*  "Center" tail definition */

        {  0,  0 },

        {  0, 76 },

        {  3, 82 },

        { 10, 84 },

        { 18, 82 },

        { 21, 76 },

        { 21, 70 },

    };

  

    XPoint              offCenterTail[N_TAIL_PTS];  /* off center tail    */

    XPoint              newTail[N_TAIL_PTS];        /*  Tail at time "t"  */

    XGCValues           bitmapGCV;                  /*  GC for drawing    */

    unsigned long       valueMask;        

    int                 i;

  

    /*

     *  Create GC for drawing tail

     */

    bitmapGCV.function       = GXcopy;

    bitmapGCV.plane_mask     = AllPlanes;

    bitmapGCV.foreground     = 1;

    bitmapGCV.background     = 0;

    bitmapGCV.line_width     = 15;

    bitmapGCV.line_style     = LineSolid;

    bitmapGCV.cap_style      = CapRound;

    bitmapGCV.join_style     = JoinRound;

    bitmapGCV.fill_style     = FillSolid;

    bitmapGCV.subwindow_mode = ClipByChildren;

    bitmapGCV.clip_x_origin  = 0;

    bitmapGCV.clip_y_origin  = 0;

    bitmapGCV.clip_mask      = None;

  

    valueMask =

        GCFunction  | GCPlaneMask     | GCForeground  | GCBackground  |

        GCLineWidth | GCLineStyle     | GCCapStyle    | GCJoinStyle   |

        GCFillStyle | GCSubwindowMode | GCClipXOrigin | GCClipYOrigin |

        GCClipMask;

  

    tailBitmap = XCreateBitmapFromData(dpy, root,

                                       tail_bits, tail_width, tail_height);

    bitmapGC = XCreateGC(dpy, tailBitmap, valueMask, &bitmapGCV);

  

    {

        /*

         *  Create an "off-center" tail to deal with the fact that

         *  the tail has a hook to it.  A real pendulum so shaped would

         *  hang a bit to the left (as you look at the cat).

         */

        angle = -0.08;

        sinTheta = sin(angle);

        cosTheta = cos(angle);

        for (i = 0; i < N_TAIL_PTS; i++) {

            offCenterTail[i].x = (int)((double)(tail[i].x) * cosTheta +

                                       (double)(tail[i].y) * sinTheta);

            offCenterTail[i].y = (int)((double)(-tail[i].x) * sinTheta +

                                       (double)(tail[i].y) * cosTheta);

        }

    }

  

    /*

     *  Compute pendulum function.

     */

    angle = A * sin(omega * t + phi);

    sinTheta = sin(angle);

    cosTheta = cos(angle);

    /*

     *  Rotate the center tail about its origin by "angle" degrees.

     */

    for (i = 0; i < N_TAIL_PTS; i++) {

        newTail[i].x = (int)((double)(offCenterTail[i].x) * cosTheta +

                             (double)(offCenterTail[i].y) * sinTheta);

        newTail[i].y = (int)((double)(-offCenterTail[i].x) * sinTheta +

                             (double)(offCenterTail[i].y) * cosTheta);

        newTail[i].x += tailOffset.x;

        newTail[i].y += tailOffset.y;

    }

    /*

     *  Create pixmap for drawing tail (and stippling on update)

     */

    XDrawLines(dpy, tailBitmap, bitmapGC,

               newTail, N_TAIL_PTS, CoordModeOrigin);

  

    XFreeGC(dpy, bitmapGC);

    return (tailBitmap);

}

  
  

static Pixmap CreateEyePixmap(t)

    double      t;                     /*  "Time" parameter  */

{

    Pixmap      eyeBitmap;

    GC          bitmapGC;

  

    double      A = 0.7;

    double      omega = 1.0;

    double      phi = 3 * M_PI_2;

    double      angle;

  

    double      u, w;                   /*  Sphere parameters    */

    float       r;                      /*  Radius               */

    float       x0, y0, z0;             /*  Center of sphere     */

  

    XPoint      pts[100];

    XGCValues           bitmapGCV;      /*  GC for drawing       */

    unsigned long       valueMask;

    int                 i, j;

  

    typedef struct {

        double    x, y, z;

    } Point3D;

  

    /*

     *  Create GC for drawing eyes

     */

    bitmapGCV.function       = GXcopy;

    bitmapGCV.plane_mask     = AllPlanes;

    bitmapGCV.foreground     = 1;

    bitmapGCV.background     = 0;

    bitmapGCV.line_width     = 15;

    bitmapGCV.line_style     = LineSolid;

    bitmapGCV.cap_style      = CapRound;

    bitmapGCV.join_style     = JoinRound;

    bitmapGCV.fill_style     = FillSolid;

    bitmapGCV.subwindow_mode = ClipByChildren;

    bitmapGCV.clip_x_origin  = 0;

    bitmapGCV.clip_y_origin  = 0;

    bitmapGCV.clip_mask      = None;

  

    valueMask =

        GCFunction  | GCPlaneMask     | GCForeground  | GCBackground  |

        GCLineWidth | GCLineStyle     | GCCapStyle    | GCJoinStyle   |

        GCFillStyle | GCSubwindowMode | GCClipXOrigin | GCClipYOrigin |

        GCClipMask;

  

    eyeBitmap = XCreateBitmapFromData(dpy, root,

                                      eyes_bits, eyes_width, eyes_height);

    bitmapGC = XCreateGC(dpy, eyeBitmap, valueMask, &bitmapGCV);

  

    /*

     *  Compute pendulum function.

     */

    w = M_PI/2.0;

  

    angle = A * sin(omega * t + phi) + w;

  

    x0 = 0.0;

    y0 = 0.0;

    z0 = 2.0;

    r  = 1.0;

  

    for (i = 0, u = -M_PI/2.0; u < M_PI/2.0; i++, u += 0.25) {

        Point3D    pt;

  

        pt.x = x0 + r * cos(u) * cos(angle + M_PI/7.0);

        pt.z = z0 + r * cos(u) * sin(angle + M_PI/7.0);

        pt.y = y0 + r * sin(u);

  

        pts[i].x = (int)(((pt.z == 0.0 ? pt.x : pt.x / pt.z) * 23.0) + 12.0);

        pts[i].y = (int)(((pt.z == 0.0 ? pt.y : pt.y / pt.z) * 23.0) + 11.0);

    }

  

    for (u = M_PI/2.0; u > -M_PI/2.0; i++, u -= 0.25) {

        Point3D    pt;

  

        pt.x = x0 + r * cos(u) * cos(angle - M_PI/7.0);

        pt.z = z0 + r * cos(u) * sin(angle - M_PI/7.0);

        pt.y = y0 + r * sin(u);

  

        pts[i].x = (int)(((pt.z == 0.0 ? pt.x : pt.x / pt.z) * 23.0) + 12.0);

        pts[i].y = (int)(((pt.z == 0.0 ? pt.y : pt.y / pt.z) * 23.0) + 11.0);

    }

  

    /*

     *  Create pixmap for drawing eye (and stippling on update)

     */

    XFillPolygon(dpy, eyeBitmap, bitmapGC, pts, i, Nonconvex, CoordModeOrigin);

  

    for (j = 0; j < i; j++) {

        pts[j].x += 31;

    }

    XFillPolygon(dpy, eyeBitmap, bitmapGC, pts, i, Nonconvex, CoordModeOrigin);

    XFreeGC(dpy, bitmapGC);

  

    return (eyeBitmap);

}

  
  

static Widget CreateToggle(label, parent)

    char   *label;

    Widget  parent;

{

    Widget      w;

    int         n;

    Arg         args[1];

    XmString    tcs;

  

    tcs = XmStringLtoRCreate(label, XmSTRING_DEFAULT_CHARSET);

  

    n = 0;

    XtSetArg(args[n], XmNlabelString, tcs);  n++;

    w = XmCreateToggleButton(parent, "toggle", args, n);

    XtManageChild(w);

  

    XmStringFree(tcs);

  

    return (w);

}

  

static Widget CreatePushButton(label, parent)

    char   *label;

    Widget  parent;

{

    Widget   w;

    int      n;

    Arg      args[1];

    XmString tcs;

  

    tcs = XmStringLtoRCreate(label, XmSTRING_DEFAULT_CHARSET);

  

    n = 0;

    XtSetArg(args[n], XmNlabelString, tcs);  n++;

    w = XmCreatePushButton(parent, "pushButton", args, n);

    XtManageChild(w);

  

    XmStringFree(tcs);

  

    return (w);

}

  
  

static void HandleExpose(w, clientData, _callData)

    Widget    w;

    XtPointer clientData;

    XtPointer _callData;

{

    XmDrawingAreaCallbackStruct *callData =

        (XmDrawingAreaCallbackStruct *)_callData;

  

    /*

     *  Ignore if more expose events for this window in the queue

     */

    if (((XExposeEvent *)(callData->event))->count > 0) {

        return;

    }

  

    /*

     *  Redraw the clock face

     */

    DrawClockFace(secondHandLength, radius);

  

    /*

     *  Call the Tick routine so that the face gets

     *  refreshed with the correct time.  However, call

     *  it with arg #2 as "False", so the ticking doesn't

     *  propagate.

     */

    Tick(w, False);

}

  
  

static void Tick(w, add)

    Widget      w;

    int         add;

{

    static Bool        beeped = False;  /*  Beeped already?        */

    time_t             timeValue;       /*  What time is it?       */

  

    /*

     *  Don't do anything when clock is iconified.  Ticking will

     *  restart when the clock is de-iconifed.

     */

    if (iconified) {

        return;

    }

  

    /*

     *  Get the time and convert.

     */

    time(&timeValue);

    tm = *localtime(&timeValue);

  

    /*

     *  If ticking is to continue, add the next timeout

     */

    if (add) {

        XtAppAddTimeOut(appContext, appData.update, Tick, w);

    }

  

    /*

     *  Beep on the half hour; double-beep on the hour.

     */

    if (appData.chime) {

        if (beeped && (tm.tm_min != 30) && (tm.tm_min != 0)) {

            beeped = 0;

        }

        if (((tm.tm_min == 30) || (tm.tm_min == 0)) && (!beeped)) {

            beeped = 1;

            XBell(dpy, 50);

            if (tm.tm_min == 0) {

                XBell(dpy, 50);

            }

        }

    }

  

    /*

     *  The second (or minute) hand is sec (or min)

     *  sixtieths around the clock face. The hour hand is

     *  (hour + min/60) twelfths of the way around the

     *  clock-face.  The derivation is left as an excercise

     *  for the reader.

     */

    /*

     *  12 hour clock.

     */

    if (tm.tm_hour > 12) {

        tm.tm_hour -= 12;

    }

  

    if (numSegs == 0 ||    tm.tm_min != otm.tm_min ||

        tm.tm_hour != otm.tm_hour) {

        segBufPtr = segBuf;

        numSegs = 0;

        DrawClockFace(secondHandLength, radius);

  

        /*

         *  Calculate the minute hand, fill it in with its

         *  color and then outline it.  Next, do the same

         *  with the hour hand.  This is a cheap hidden

         *  line algorithm.

         */

        DrawHand(minuteHandLength, handWidth,

             ((double) tm.tm_min)/60.0);

        if (appData.handColor != appData.background) {

            XFillPolygon(dpy,

                         clockWindow, handGC,

                         segBuf, VERTICES_IN_HANDS + 2,

                         Convex, CoordModeOrigin);

        }

        XDrawLines(dpy,

                   clockWindow, highGC,

                   segBuf, VERTICES_IN_HANDS + 2,

                   CoordModeOrigin);

        DrawHand(hourHandLength, handWidth,

                ((((double)tm.tm_hour) +

                  (((double)tm.tm_min) / 60.0)) / 12.0));

        if (appData.handColor != appData.background) {

            XFillPolygon(dpy,

                         clockWindow, handGC,

                         &(segBuf[VERTICES_IN_HANDS + 2]),

                         VERTICES_IN_HANDS + 2,

                         Convex, CoordModeOrigin);

        }

  

        XDrawLines(dpy,

                   clockWindow, highGC,

                   &(segBuf[VERTICES_IN_HANDS + 2]),

                   VERTICES_IN_HANDS + 2,

                   CoordModeOrigin);

    }

    else {

          UpdateEyesAndTail();

    }

    otm = tm;

    XSync(dpy, False);

}

  
  

static void HandleInput(w, clientData, _callData)

    Widget     w;

    XtPointer  clientData;

    XtPointer  _callData;

{

    XmDrawingAreaCallbackStruct *callData =

        (XmDrawingAreaCallbackStruct *)_callData;

    int           n;

    Arg           args[10];

    static int    menuInited = False;

    static Widget menuShell, menu, setW, bellW, chimeW,

                  ackW, rereadW, editW, exitW;

    static Widget sepW1, sepW2;

  

    if (callData->event->type != ButtonPress) {

        return;

    }

    if (((XButtonEvent *)(callData->event))->button != Button1) {

        return;

    }

    if (!menuInited) {

        menuInited = True;

        n = 0;

        XtSetArg(args[n], XmNwidth,  100);    n++;

        XtSetArg(args[n], XmNheight, 100);    n++;

        menuShell = XmCreateMenuShell(w, "menuShell", args, n);

        n = 0;

        XtSetArg(args[n], XmNrowColumnType, XmMENU_POPUP);    n++;

        XtSetArg(args[n], XmNwhichButton,   1);            n++;

        menu = XmCreateRowColumn(menuShell, "menu", args, n);

        n = 0;

        sepW1 = XmCreateSeparator(menu, "sepW1", args, n);

        XtManageChild(sepW1);

        setW = CreateToggle("Alarm Set", menu);

        n = 0;

        XtSetArg(args[n], XmNset, appData.alarmSet ? True : False);    n++;

        XtSetValues(setW, args, n);

        XtAddCallback(setW, XmNvalueChangedCallback,

                      AlarmSetCallback, NULL);

        bellW = CreateToggle("Alarm Bell", menu);

        n = 0;

        XtSetArg(args[n], XmNset, appData.alarmBell ?  True : False);    n++;

        XtSetValues(bellW, args, n);

        XtAddCallback(bellW, XmNvalueChangedCallback,

                      AlarmBellCallback, NULL);

        chimeW = CreateToggle("Chime", menu);

        n = 0;

        XtSetArg(args[n], XmNset, appData.chime ?  True : False);    n++;

        XtSetValues(chimeW, args, n);

        XtAddCallback(chimeW, XmNvalueChangedCallback,

                      ChimeCallback, NULL);

        ackW = CreatePushButton("Acknowledge Alarm", menu);

        XtAddCallback(ackW, XmNactivateCallback,

                      AckAlarmCallback, NULL);

        rereadW = CreatePushButton("Reread Alarm File", menu);

        XtAddCallback(rereadW, XmNactivateCallback,

                      RereadAlarmCallback, setW);

  

        editW = CreatePushButton("Edit Alarm File", menu);

        XtAddCallback(editW, XmNactivateCallback,

                      EditAlarmCallback, (XtPointer)setW);

        n = 0;

        sepW2 = XmCreateSeparator(menu, "sepW2", args, n);

        XtManageChild(sepW2);

        exitW = CreatePushButton("Exit", menu);

        XtAddCallback(exitW, XmNactivateCallback, ExitCallback, NULL);

    }

    XmMenuPosition(menu, (XButtonPressedEvent *)(callData->event));

    XtManageChild(menu);

}

  
  

static void AlarmSetCallback(w, clientData, callData)

    Widget     w;

    XtPointer  clientData;

    XtPointer  callData;

{

    if ((appData.alarmSet = appData.alarmSet ? False : True)) {

        SetAlarm(appData.alarmFile);

    } else {

        if (alarmOn) {

            AlarmOff();

            DrawClockFace(secondHandLength, radius);

        }

        SetAlarm(NULL);

    }

}

  
  

static void AlarmBellCallback(w, clientData, callData)

     Widget     w;

     XtPointer  clientData;

     XtPointer  callData;

{

    SetBell((appData.alarmBell = appData.alarmBell ? False : True) ?

            appData.alarmBellPeriod : 0);

}

  
  

static void ChimeCallback(w, clientData, callData)

    Widget     w;

    XtPointer  clientData;

    XtPointer  callData;

{

    appData.chime = appData.chime ? False : True;

}

  
  

static void AckAlarmCallback(w, clientData, callData)

    Widget     w;

    XtPointer  clientData;

    XtPointer  callData;

{

    if (alarmOn) {

        AlarmOff();

        SetAlarm(appData.alarmSet ? appData.alarmFile : NULL);

        DrawClockFace(secondHandLength, radius);    

    }

}

  
  

static void RereadAlarmCallback(w, clientData, callData)

    Widget     w;

    XtPointer  clientData;

    XtPointer  callData;

{

    Widget     setW = (Widget)clientData;

  

    int        n;

    Arg        args[1];

  

    appData.alarmSet = True;

  

    if (alarmOn) {

        AlarmOff();

        DrawClockFace(secondHandLength, radius);

    }

  

    SetAlarm(appData.alarmFile);

  

    n = 0;

    XtSetArg(args[n], XmNset, True);    n++;

    XtSetValues(setW, args, n);

}

  
  

static void EditAlarmCallback(w, clientData, callData)

    Widget     w;

    XtPointer  clientData;

    XtPointer  callData;

{

    Widget       setW = (Widget)clientData;

  

    char        *editor;

    char         cmdString[80];

  

    editor = getenv("EDITOR");

  

    if (editor == NULL) {

        strcpy(cmdString, "xterm -e vi ");

        strcat(cmdString, " \0");

#ifdef HAS_GNU_EMACS

    } else if (strcmp(editor, "emacs") == 0) {

        strcpy(cmdString, "emacs ");

        strcat(cmdString, " ");

#else

    } else if (strcmp(editor, "emacs") == 0) {

        strcpy(cmdString, "xterm -e emacs ");

        strcat(cmdString, " ");

#endif  /*  HAS_GNU_EMACS  */

    } else if (strcmp(editor, "vi") == 0) {

        strcpy(cmdString, "xterm -e vi ");

        strcat(cmdString, " ");

    } else {

        strcpy(cmdString, "xterm -e ");

        strcat(cmdString, " ");

        strcat(cmdString, editor);

        strcat(cmdString, " ");

    }

  

    strcat(cmdString, appData.alarmFile);

  

    if (system(cmdString) != 127) {

        RereadAlarmCallback(w, setW, callData);

    }

}

  
  

static void ExitCallback(w, clientData, callData)

    Widget     w;

    XtPointer  clientData;

    XtPointer  callData;

{

    exit(0);

}

  

static void EraseHands(w, tm)

    Widget      w;

    struct tm   *tm;

{

    if (numSegs > 0) {

        if (showSecondHand) {

            XDrawLines(dpy, clockWindow, eraseGC,

                       &(segBuf[2 * (VERTICES_IN_HANDS + 2)]),

                       VERTICES_IN_HANDS * 2 - 1,

                       CoordModeOrigin);

  

            if (appData.handColor != appData.background) {

                XFillPolygon(dpy, clockWindow, eraseGC,

                             &(segBuf[2 * (VERTICES_IN_HANDS + 2)]),

                             VERTICES_IN_HANDS * 2 - 1,

                             Convex, CoordModeOrigin);

            }

        }

        if (!tm || tm->tm_min != otm.tm_min || tm->tm_hour != otm.tm_hour) {

            XDrawLines(dpy, clockWindow, eraseGC,

                       segBuf, VERTICES_IN_HANDS + 2,

                       CoordModeOrigin);

  

            XDrawLines(dpy, clockWindow, eraseGC,

                       &(segBuf[VERTICES_IN_HANDS + 2]), VERTICES_IN_HANDS,

                       CoordModeOrigin);

  

            if (appData.handColor != appData.background) {

                XFillPolygon(dpy, clockWindow, eraseGC,

                             segBuf, VERTICES_IN_HANDS + 2,

                             Convex, CoordModeOrigin);

  

                XFillPolygon(dpy, clockWindow, eraseGC,

                             &(segBuf[VERTICES_IN_HANDS + 2]),

                             VERTICES_IN_HANDS + 2,

                             Convex, CoordModeOrigin);

            }

        }

    }

}

  
  

static void MapCallback(w, clientData, event, continueToDispatch)

    Widget      w;

    XtPointer   clientData;

    XEvent     *event;

    Boolean    *continueToDispatch;

{

    *continueToDispatch = True;

  

    if (event->type == MapNotify) {

        iconified = False;

        XSync(dpy, False);

        Tick(XtWindowToWidget(dpy, clockWindow), True);

    }

    else if (event->type == UnmapNotify) {

        iconified = True;

    }

}