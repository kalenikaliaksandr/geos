---
title: "C API Programming"
date: 2021-10-04T14:21:00-07:00
draft: false
weight: 40
---

Most programs using GEOS use the C API, rather than building against the C++ headers. The C API offers the a number of benefits:

* Stable API, that preserves behaviour and function naming over multiple releases.
* Stable ABI, allowing new binaries to be dropped into place without requiring a rebuild of dependent applications.
* Simple access pattern, using the [simple features model](https://en.wikipedia.org/wiki/Simple_Features) as the basis for most operations.

In exchange for this simplicity and stability, the C API has a few requirements from application authors:

* Explicit memory management. If you create a GEOS object with a GEOS function, you have to remember to free it using the appropriate GEOS destructor.

The C API is entirely contained in the [geos_c.h](../../doxygen/geos__c_8h.html) header file.

## Building a Program

The simplest GEOS C API application needs to include the API header, declare a message handler, initialize the GEOS globals, and link to the GEOS C library when built.

```c
/* geos_hello_world.c */

#include <stdio.h>  /* for printf */
#include <stdarg.h> /* for va_list */

/* Only the CAPI header is required */
#include <geos_c.h>

/*
* GEOS requires two message handlers to return
* error and notice message to the calling program.
*
*   typedef void(* GEOSMessageHandler) (const char *fmt,...)
*
* Here we stub out an example that just prints the
* messages to stdout.
*/
static void
geos_msg_handler(const char* fmt, ...)
{
    va_list ap;
    va_start(ap, fmt);
    vprintf (fmt, ap);
    va_end(ap);
}

int main()
{
    /* Send notice and error messages to the terminal */
    initGEOS(geos_msg_handler, geos_msg_handler);

    /* Read WKT into geometry object */
    GEOSWKTReader* reader = GEOSWKTReader_create();
    GEOSGeometry* geom_a = GEOSWKTReader_read(reader, "POINT(1 1)");

    /* Convert result to WKT */
    GEOSWKTWriter* writer = GEOSWKTWriter_create();
    char* wkt = GEOSWKTWriter_write(writer, geom_a);
    printf("Geometry: %s\n", wkt);

    /* Clean up allocated objects */
    GEOSWKTReader_destroy(reader);
    GEOSWKTWriter_destroy(writer);
    GEOSGeom_destroy(geom_a);
    GEOSFree(wkt);

    /* Clean up the global context */
    finishGEOS();
    return 0;
}
```

When compiling the program, remember to link in the GEOS C library.

```bash
cc geos_hello_world.c -o geos_hello_world -l geos_c
```

## Reentrant/Threadsafe API

Every function in the examples above and below is shadowed by a reentrant function carrying a `_r` suffix. The reentrant functions work the same as their simple counterparts, but they all have one extra parameter, a `GEOSContextHandle_t`.

The `GEOSContextHandle_t` carries a thread-local state that is equivalent to the state initialized by the `initGEOS()` call in the simple example above.

To use the reentrant API, you skip calling `initGEOS()` and instead call `GEOS_init_r()` to create a context local to your thread. Each thread that will be running GEOS operations should create its own context prior to working with the GEOS API.

```c
/* geos_hello_world.c */

#include <stdio.h>  /* for printf */
#include <stdarg.h> /* for va_list */

/* Only the CAPI header is required */
#include <geos_c.h>

static void
geos_msg_handler(const char* fmt, ...)
{
    va_list ap;
    va_start(ap, fmt);
    vprintf (fmt, ap);
    va_end(ap);
}

int main()
{
    /* Send notice and error messages to the terminal */
    GEOSContextHandle_t ctx = GEOS_init_r ();
    GEOSContext_setNoticeHandler_r(ctx, geos_msg_handler);
    GEOSContext_setErrorHandler_r(ctx, geos_msg_handler);

    /* Read WKT into geometry object */
    GEOSWKTReader* reader = GEOSWKTReader_create_r(ctx);
    GEOSGeometry* geom_a = GEOSWKTReader_read_r(ctx, reader, "POINT(1 1)");

    /* Convert result to WKT */
    GEOSWKTWriter* writer = GEOSWKTWriter_create_r(ctx);
    char* wkt = GEOSWKTWriter_write_r(ctx, writer, geom_a);
    printf("Geometry: %s\n", wkt);

    /* Clean up allocated objects */
    GEOSWKTReader_destroy_r(ctx, reader);
    GEOSWKTWriter_destroy_r(ctx, writer);
    GEOSGeom_destroy_r(ctx, geom_a);
    GEOSFree_r(ctx, wkt);

    /* Clean up the global context */
    GEOS_finish_r(ctx);
    return 0;
}
```

Note that the overall structure of the code is identical, but the reentrant variants are used, and the preamble and cleanup are slightly different.

## Object Model

The `GEOSCoordSequence` and `GEOSGeometry` objects are at the heart of the GEOS object model.

### GEOSCoordSequence

`GEOSCoordSequence` is an ordered list of coordinates (2 or 3 dimensional). There are a number of ways to make a `GEOSCoordSequence`.

You can create a `GEOSCoordSequence` by creating a blank one and then setting the coordinate values.

```c
double xList[] = {1.0, 2.0, 3.0};
double yList[] = {3.0, 2.0, 1.0};
size_t seqSize = 3;
size_t seqDims = 2;

GEOSCoordSequence* seq = GEOSCoordSeq_create(seqSize, seqDims);

for (size_t i = 0; i < seqSize; i++) {
    seq->setXY(i, xList[i], yList[i]);
}
GEOSCoordSeq_destroy(seq);
```

You can also create a `GEOSCoordSequence` and populate it simultaneously from coordinate arrays.

```c
double xList[] = {1.0, 2.0, 3.0};
double yList[] = {3.0, 2.0, 1.0};
size_t seqSize = 3;

GEOSCoordSequence* seq = GEOSCoordSeq_copyFromArrays(
    xList,
    yList,
    NULL,  /* Zs */
    NULL,  /* Ms */
    seqSize);

GEOSCoordSeq_destroy(seq);
```

Finally, you can create a `GEOSCoordSequence` and populate it simultaneously directly from a single coordinate buffer (an array of double in coordinate order, eg: `XYXYXYX`).

```c
/* Coordinates in a buffer (X,Y, X,Y, X,Y) */
double coordBuf[] = {1.0,3.0, 2.0,2.0, 3.0,1.0};
size_t seqSize = 3;

GEOSCoordSequence* seq = GEOSCoordSeq_copyFromBuffer(
    coordBuf,
    seqSize,
    0, /* hasZ */
    0  /* hasM */
    );

GEOSCoordSeq_destroy(seq);
```

Note that while you can reclaim the memory for a `GEOSCoordSequence` directly using `GEOSCoordSeq_destroy()`, you usually **will not have to** since creating a `GEOSGeometry` with a `GEOSCoordSequence` hands ownership of the sequence to the new geometry.

When writing data back from GEOS to whatever application you are using, you have the option of using a standard serialization format like WKB (see below) or by writing back to arrays or buffers.

* GEOSCoordSeq_copyToArrays()
* GEOSCoordSeq_copyToBuffer()

Using the array or buffer methods can often be **faster** than using direct coordinate reading or serialization formats, if the target structures use coordinate arrays or XY binary buffers.

### GEOSGeometry

The workhorse of the GEOS C API is the `GEOSGeometry`.  `GEOSGeometry` can be a point, linestring, polygon, multipoint, multilinestring, multipolygon, or geometrycollection. Most functions in the GEOS C API have a `GEOSGeometry` as a parameter or return type. Clean up `GEOSGeometry` using `GEOSGeom_destroy()`.

There are many constructors for `GEOSGeometry`:

* GEOSGeom_createPoint()
* GEOSGeom_createPointFromXY()
* GEOSGeom_createLinearRing()
* GEOSGeom_createLineString()
* GEOSGeom_createPolygon()
* GEOSGeom_createCollection()
* GEOSGeom_createEmptyPoint()
* GEOSGeom_createEmptyLineString()
* GEOSGeom_createEmptyPolygon()
* GEOSGeom_createEmptyCollection()

The "createEmpty" functions take no arguments are return geometries that are "empty", that is they represent an empty set of space. The intersection of two **disjoint polygons** is a "empty polygon", for example.

The `GEOSGeom_createPoint()`, `GEOSGeom_createLinearRing()` and `GEOSGeom_createLinearRing()` all take in a single `GEOSCoordSequence` and take ownership of that sequence, so freeing the geometry with `GEOSGeom_destroy()` frees all memory.

```c
double coordBuf[] = {1.0,3.0, 2.0,2.0, 3.0,1.0};
size_t seqSize = 3;
GEOSCoordSequence* seq = GEOSCoordSeq_copyFromBuffer(
    coordBuf, seqSize, 0, 0);

/* Takes ownership of sequence */
GEOSGeometry* geom = GEOSGeom_createLineString(seq);

/* Frees all memory */
GEOSGeom_destroy(geom);
```

The `GEOSGeom_createPolygon()` and `GEOSGeom_createCollection()` functions both require an array of inputs:

* an array of inner ring `GEOSCoordSequence` to create polygons; and,
* an array of `GEOSGeometry` to create collections.

As in the other creation functions, ownership of the contained objects is transferred to the new geometry. However, ownership of the array that **holds** the contained objects is not transferred.

```c
/* Two points in an array */
size_t npoints = 2;
GEOSGeometry** points = malloc(sizeof(GEOSGeometry*) * npoints);
points[0] = GEOSGeom_createPointFromXY(0.0, 0.0);
points[1] = GEOSGeom_createPointFromXY(0.0, 0.0);

/* takes ownership of the points in the array */
/* but not the array itself */
GEOSGeometry* collection = GEOSGeom_createCollection(
    GEOS_MULTIPOINT,  /* collection type */
    points,           /* geometry array */
    npoints);

/* frees collection and contained points */
GEOSGeom_destroy(collection);

/* frees the containing array */
free(points);
```

### Readers and Writers

The examples above build `GEOSCoordSequence` from arrays of double, and `GEOSGeometry` from coordinate sequences, but it is also possible to directly read from and write to standard geometry formats:

* Well-Known Text ([WKT]({{< ref "/specifications/wkt" >}}))
* Well-Known Binary ([WKB]({{< ref "/specifications/wkb" >}}))
* [GeoJSON](https://datatracker.ietf.org/doc/html/rfc7946)

For example, reading and writing WKT:

```c
const char* wkt_in = "POLYGON((0 0, 10 0, 10 10, 0 10, 0 0))";

/* Read the WKT into geometry object */
GEOSWKTReader* reader = GEOSWKTReader_create();
GEOSGeometry* geom = GEOSWKTReader_read(reader, wkt_in);

/* Convert geometry back to WKT */
GEOSWKTWriter* writer = GEOSWKTWriter_create();
/* Trim trailing zeros off output */
GEOSWKTWriter_setTrim(writer, 1);
char* wkt_out = GEOSWKTWriter_write(writer, geom);

/* Clean up everything we allocated */
GEOSWKTReader_destroy(reader);
GEOSGeom_destroy(geom);
/* Use GEOSFree() to free memory allocated inside GEOS~ */
GEOSFree(wkt_out);
```

Note that the output WKT string is freed using `GEOSFree()`, not the system `free()`. This ensures that the same library that allocates the memory also frees it, which is important for some platforms (Windows primarily).

For more information about the specific options available for each format, see the documentation for the various readers and writers.

* GEOSWKTReader / GEOSWKTWriter
* GEOSWKBReader / GEOSWKBWriter
* GEOSGeoJSONReader / GEOSGeoJSONWriter

For a complete example using a reader and writer, see [capi_read.c](https://github.com/libgeos/geos/blob/main/examples/capi_read.c).

### Prepared Geometry

The GEOS "prepared geometry" is conceptually similar to a database "prepared statement": by doing a little up-front work to set-up a handler, you can reap a performance benefit when you execute repeated function calls on that object.

Prepared geometries contain internal indexes that make calls to the "spatial predicate" functions like `GEOSPreparedIntersects()` and `GEOSPreparedContains()` much much faster. These are functions that take in two geometries and return true or false.

If you are going to be making repeated calls to predicates on the same geometry, using a prepared geometry could be a big performance boost, at the cost of a little extra complexity.

```c
/* One concave polygon */
const char* wkt = "POLYGON ((189 115, 200 170, 130 170, 35 242, 156 215, 210 290, 274 256, 360 190, 267 215, 300 50, 200 60, 189 115))";

/* Read the WKT into geometry objects */
GEOSWKTReader* reader = GEOSWKTReader_create();
GEOSGeometry* geom = GEOSWKTReader_read(reader, wkt);
GEOSWKTReader_destroy(reader);

/* Prepare the geometry */
const GEOSPreparedGeometry* prep_geom = GEOSPrepare(geom);

/* Make a point to test */
GEOSGeometry* pt = GEOSGeom_createPointFromXY(190, 200);

/* Check if the point and polygon intersect */
if (GEOSPreparedIntersects(prep_geom, pt)) {
    /* done something ... */
}

/* Note that both prepared and original geometry are destroyed */
GEOSPreparedGeom_destroy(prep_geom);
GEOSGeom_destroy(geom);
GEOSGeom_destroy(pt);
```

For a complete example of using prepared geometry to accelerate multiple predicate tests, see the [capi_prepared.c](https://github.com/libgeos/geos/blob/main/examples/capi_prepared.c) example.


### STRTree Index

* GEOSSTRtree

