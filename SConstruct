import os
import sys
import glob
import excons
import SCons.Script # pylint: disable=import-error

env = excons.MakeBaseEnv()

out_topdir = excons.OutputBaseDirectory()
out_incdir = out_topdir + "/include"
out_libdir = out_topdir + "/lib"

static = (excons.GetArgument("lcms2-static", 1, int) != 0)
suffix = excons.GetArgument("lcms2-suffix", "")


def LCMS2Name():
   name = "lcms2" + suffix
   if sys.platform == "win32" and static:
      name = "lib" + name
   return name

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
   excons.Link(env, LCMS2Path(), static=static, force=True, silent=True)
   if sys.platform != "win32":
      env.Append(LIBS=["m"])


lcms_defs = []
if not static:
   lcms_defs.append("CMS_DLL")


tiff_deps = []
tiff_opts = {}


# Jpeg is required both as a direct dependency for jpgicc tool and by libtiff
def JpegLibname(static):
   return "jpeg"

rv = excons.ExternalLibRequire("libjpeg", libnameFunc=JpegLibname)
if not rv["require"]:
   excons.PrintOnce("LCMS2: Build libjpeg from source ...")
   excons.Call("libjpeg-turbo", targets=["libjpeg"], imp=["LibjpegName", "LibjpegPath", "RequireLibjpeg"])
   jpegstatic = (excons.GetArgument("libjpeg-static", 1, int) != 0)
   if sys.platform == "win32":
      tiff_deps.append(excons.cmake.OutputsCachePath("libjpeg"))
   else:
      tiff_deps.append(excons.automake.OutputsCachePath("libjpeg"))
   tiff_opts["with-libjpeg"] = os.path.dirname(os.path.dirname(LibjpegPath(jpegstatic))) # pylint: disable=undefined-variable
   tiff_opts["libjpeg-static"] = (1 if jpegstatic else 0)
   tiff_opts["libjpeg-name"] = LibjpegName(jpegstatic) # pylint: disable=undefined-variable
   def JpegRequire(env):
      RequireLibjpeg(env, static=jpegstatic) # pylint: disable=undefined-variable
else:
   JpegRequire = rv["require"]

# libtiff is required as a direct dependency for tificc tool
def TiffLibname(static):
   return "tiff"

rv = excons.ExternalLibRequire("libtiff", libnameFunc=TiffLibname)
if not rv["require"]:
   excons.PrintOnce("LCMS2: Build libtiff from source ...")
   excons.cmake.AddConfigureDependencies("libtiff", tiff_deps)
   excons.Call("libtiff", targets=["libtiff"], overrides=tiff_opts, imp=["LibtiffName", "LibtiffPath", "RequireLibtiff"])
   def TiffRequire(env):
      RequireLibtiff(env) # pylint: disable=undefined-variable
else:
   TiffRequire = rv["require"]



prjs = [
   {  "name": LCMS2Name(),
      "alias": "lcms2",
      "type": ("staticlib" if static else "sharedlib"),
      "version": "2.10.0",
      "symvis": "default",
      "soname": "lib" + LCMS2Name() + ".so.2",
      "install_name": "lib" + LCMS2Name() + ".2.dylib",
      "defs": lcms_defs + (["CMS_DLL_BUILD"] if not static else []),
      "srcs": glob.glob("src/*.c"),
      "install": {"include": glob.glob("include/*.h")}
   },
   {  "name": "lcms2_tools_common",
      "type": "staticlib",
      "desc": "Utility library for command line tools",
      "defs": lcms_defs,
      "srcs": glob.glob("utils/common/*.c")
   },
   {  "name": "jpgicc",
      "alias": "lcms2-tools",
      "type": "program",
      "incdirs": ["utils/common"],
      "srcs": glob.glob("utils/jpgicc/*.c"),
      "staticlibs": ["lcms2_tools_common"],
      "custom": [RequireLCMS2, JpegRequire]
   },
   {  "name": "tificc",
      "type": "program",
      "alias": "lcms2-tools",
      "incdirs": ["utils/common"],
      "srcs": ["utils/tificc/tificc.c"],
      "staticlibs": ["lcms2_tools_common"],
      "custom": [RequireLCMS2, TiffRequire]
   },
   {  "name": "linkicc",
      "type": "program",
      "alias": "lcms2-tools",
      "incdirs": ["utils/common"],
      "srcs": glob.glob("utils/linkicc/*.c"),
      "staticlibs": ["lcms2_tools_common"],
      "custom": [RequireLCMS2]
   },
   {  "name": "psicc",
      "type": "program",
      "alias": "lcms2-tools",
      "incdirs": ["utils/common"],
      "srcs": glob.glob("utils/psicc/*.c"),
      "staticlibs": ["lcms2_tools_common"],
      "custom": [RequireLCMS2]
   },
   {  "name": "transicc",
      "type": "program",
      "alias": "lcms2-tools",
      "incdirs": ["utils/common"],
      "srcs": glob.glob("utils/transicc/*.c"),
      "staticlibs": ["lcms2_tools_common"],
      "custom": [RequireLCMS2]
   }
]

excons.AddHelpOptions(lcms2="""LCMS2 OPTIONS
  lcms2-static=0|1   : Toggle between static and shared library build [1]
  lcms2-suffix=<str> : Library suffix                                 []""")
excons.AddHelpTargets({"lcms2-tools": "jpgicc, tificc, linkicc, psicc, transicc"})

excons.DeclareTargets(env, prjs)

SCons.Script.Export("LCMS2Name LCMS2Path RequireLCMS2")
