import os
import sys
import glob
import excons

env = excons.MakeBaseEnv()

out_topdir = excons.OutputBaseDirectory()
out_incdir = out_topdir + "/include"
out_libdir = out_topdir + "/lib"

static = (excons.GetArgument("lcms2-static", 1, int) != 0)
suffix = excons.GetArgument("lcms2-suffix", "_s" if static else "")


def LCMS2Name():
   return ("lcms2" + suffix)

def LCMS2Path():
   if sys.platform == "win32":
      libname = LCMS2Name() + ".lib"
   else:
      libname = "lib" + LCMS2Name()
      libname += (".a" if static else excons.SharedLibraryLinkExt())
   return out_libdir + "/" + libname

def RequireLCMS2(env):
   if not static:
      env.Append(CPPDEFINES=["CMS_DLL"])
   env.Append(CPPPATH=[out_incdir])
   env.Append(LIBPATH=[out_libdir])
   excons.Link(env, LCMS2Name(), static=static, force=True, silent=True)
   if sys.platform != "win32":
      env.Append(LIBS=["m"])


lcms_defs = []
if not static:
   lcms_defs.append("CMS_DLL")


jpgicc_deps = []
tificc_deps = []
tiff_deps = []
tiff_opts = {}

build_tiff = False
build_jpeg = False

for tgt in COMMAND_LINE_TARGETS:
   if tgt == "lcms2-tools":
      build_jpeg = True
      build_tiff = True
   elif tgt in ("libtiff", "tificc"):
      build_tiff = True
   elif tgt in ("libjpeg", "jpgicc"):
      build_jpeg = True


# Zlib is required by libtiff
def ZlibName(static):
   return ("z" if sys.platform != "win32" else ("zlib" if static else "zdll"))

def ZlibDefines(static):
   return ([] if static else ["ZLIB_DLL"])

rv = excons.ExternalLibRequire("zlib", libnameFunc=ZlibName, definesFunc=ZlibDefines)
if not rv["require"]:
   if build_tiff:
      excons.PrintOnce("Build zlib from source ...")
      excons.Call("zlib", imp=["ZlibName", "ZlibPath"])
      zlibstatic = (excons.GetArgument("zlib-static", 1, int) != 0)
      tiff_deps.append(excons.cmake.OutputsCachePath("zlib"))
      tiff_opts["with-zlib"] = os.path.dirname(os.path.dirname(ZlibPath(zlibstatic)))
      tiff_opts["zlib-static"] = (1 if zlibstatic else 0)
      tiff_opts["zlib-name"] = ZlibName(zlibstatic)

# Jbig is require by libtiff
rv = excons.ExternalLibRequire("jbig")
if not rv["require"]:
   if build_tiff:
      excons.PrintOnce("Build jbig from source ...")
      excons.Call("jbigkit", imp=["JbigName", "JbigPath"])
      tiff_deps.append(JbigPath())
      tiff_opts["with-jbig"] = os.path.dirname(os.path.dirname(JbigPath()))
      tiff_opts["jbig-static"] = 1
      tiff_opts["jbig-name"] = JbigName()

# Jpeg is required both as a direct dependency and by libtiff
rv = excons.ExternalLibRequire("libjpeg")
if not rv["require"]:
   if build_jpeg or build_tiff:
      excons.PrintOnce("Build libjpeg from source ...")
      excons.Call("libjpeg-turbo", imp=["LibjpegName", "LibjpegPath", "RequireLibjpeg"])
      jpegstatic = (excons.GetArgument("libjpeg-static", 1, int) != 0)
      def JpegRequire(env):
         RequireLibjpeg(env, static=jpegstatic)
      jpgicc_deps = ["libjpeg"]
      if sys.platform == "win32":
         tiff_deps.append(excons.cmake.OutputsCachePath("libjpeg"))
      else:
         tiff_deps.append(excons.automake.OutputsCachePath("libjpeg"))
      tiff_opts["with-libjpeg"] = os.path.dirname(os.path.dirname(LibjpegPath(jpegstatic)))
      tiff_opts["libjpeg-static"] = (1 if jpegstatic else 0)
      tiff_opts["libjpeg-name"] = LibjpegName(jpegstatic)
   else:
      def JpegRequire(env):
         pass
else:
   JpegRequire = rv["require"]

# libtiff is required as a direct dependency
rv = excons.ExternalLibRequire("libtiff")
if not rv["require"]:
   if build_tiff:
      excons.PrintOnce("Build libtiff from source ...")
      excons.cmake.AddConfigureDependencies("libtiff", tiff_deps)
      excons.Call("libtiff", overrides=tiff_opts, imp=["LibtiffName", "LibtiffPath", "RequireLibtiff"])
      def TiffRequire(env):
         RequireLibtiff(env)
      tificc_deps = ["libtiff"]
   else:
      def TiffRequire(env):
         pass
else:
   TiffRequire = rv["require"]


prjs = [
   {  "name": LCMS2Name(),
      "alias": "lcms2",
      "type": ("staticlib" if static else "sharedlib"),
      "version": "2.8.0",
      "symvis": "default",
      "soname": "lib" + LCMS2Name() + ".so.2",
      "install_name": "lib" + LCMS2Name() + ".2.dylib",
      "defines": lcms_defs + (["CMS_DLL_BUILD"] if not static else []), 
      "srcs": glob.glob("src/*.c"),
      "install": {"include": glob.glob("include/*.h")}
   },
   {  "name": "lcms2_tools_common",
      "type": "staticlib",
      "defines": lcms_defs,
      "srcs": glob.glob("utils/common/*.c")
   },
   {  "name": "jpgicc",
      "alias": "lcms2-tools",
      "type": "program",
      "incdirs": ["utils/common"],
      "srcs": glob.glob("utils/jpgicc/*.c"),
      "staticlibs": ["lcms2_tools_common"],
      "deps": ["lcms2"] + jpgicc_deps,
      "custom": [RequireLCMS2, JpegRequire]
   },
   {  "name": "tificc",
      "type": "program",
      "alias": "lcms2-tools",
      "incdirs": ["utils/common"],
      "srcs": ["utils/tificc/tificc.c"],
      "staticlibs": ["lcms2_tools_common"],
      "deps": ["lcms2"] + tificc_deps,
      "custom": [RequireLCMS2, TiffRequire]
   },
   {  "name": "linkicc",
      "type": "program",
      "alias": "lcms2-tools",
      "incdirs": ["utils/common"],
      "srcs": glob.glob("utils/linkicc/*.c"),
      "staticlibs": ["lcms2_tools_common"],
      "deps": ["lcms2"],
      "custom": [RequireLCMS2]
   },
   {  "name": "psicc",
      "type": "program",
      "alias": "lcms2-tools",
      "incdirs": ["utils/common"],
      "srcs": glob.glob("utils/psicc/*.c"),
      "staticlibs": ["lcms2_tools_common"],
      "deps": ["lcms2"],
      "custom": [RequireLCMS2]
   },
   {  "name": "transicc",
      "type": "program",
      "alias": "lcms2-tools",
      "incdirs": ["utils/common"],
      "srcs": glob.glob("utils/transicc/*.c"),
      "staticlibs": ["lcms2_tools_common"],
      "deps": ["lcms2"],
      "custom": [RequireLCMS2]
   }
]

excons.AddHelpOptions(lcms2="""LCMS2 OPTIONS
  lcms2-static=0|1   : Toggle between static and shared library build [1]
  lcms2-suffix=<str> : Library suffix                                 ['_s' if static, '' otherwise]""")

excons.DeclareTargets(env, prjs)

Export("LCMS2Name LCMS2Path RequireLCMS2")

Default(["lcms2"])

