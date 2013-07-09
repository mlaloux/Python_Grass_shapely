**Automatic 3D geological boreholes representation(automate v.extrude from a table ?): my solution in Python**
--------------------------------------------------------

Is there a way to automate a script with v.extrude?
Let me explain, I create boreholes in 3D and I have to make sections according to the layers to generate surfaces

![](http://osgeo-org.1560.x6.nabble.com/file/n4978722/sondseulok.png)   ![](http://osgeo-org.1560.x6.nabble.com/file/n4978722/sondidwok.png)


 ->

from a table
Borehole z layer A layer B, layer C

My solution, for the moment, is
1) Split the original vector/table in n individuals boreholes vectors/tables
2) borehole by borehole, by sections

    v.extrude input="borehole1" output="layerA1" zshift=value_layer_A (by hand) height= value_layer_B (from field of the table)
    v.extrude input="borehole1" output="layerB1" zshift=value_layer_B (by hand) height= value_layer_C (from field of the table)

    v.extrude input="borehole2" output="layerA2" zshift=value_layer_A (by hand) height= value_layer_B (from field of the table)
    v.extrude input="borehole1" output="layerB2" zshift=value_layer_B (by hand) height= value_layer_C (from field of the table)
and so

As you can see, the problems are:

1) n vectors/tables instead of one vector/table
2) "by hand", it would be more interesting to get the value directly from the table

my idea of ​​algorithm is

    for boreholes in the table:
         extract values values A,B...,n from borehole i
         the v.extrude procedures

1) I create a buffer around each point locating a borehole to obtain an area layer (circular)

![](http://osgeo-org.1560.x6.nabble.com/attachment/4978801/1/boreholesa.png)

2) I process this layer in Python (It is also possible to run it in the Python shell of the Layer Manager)

    # import modules (for the pure Python script)
    import sys, os, numpy, argparse
    import grass.script as g
    import grass.script.setup as gsetup 
    # init
    gisbase = os.environ['GISBASE']
    gisdb="/Users/me/grassdata"
    location="geol"
    mapset="mymapset"
    gsetup.init(gisbase, gisdb, location, mapset)

    # read the depths of the limits of two formations (A and B) in the boreholes table
    inmap = "boreholes" # the area layer
    val = g.read_command('v.db.select', map = inmap, columns = "A,B",fs=',', flags = 'c')
    levels=val.split('\n')
    levels=[i.split(',') for i in levels]
    print levels
    [['890', '850'], ['1040', '830'],....]

    # read the identification code of the boreholes in the table
    test= g.read_command('v.db.select', map = inmap, columns = "IDENT",fs=',', flags = 'c')
    ident =  test.split('\n')
    ident=[i.split(',') for i in ident]
    print ident
    [['3930148'], ['3930243'],.....]  
    # read info from the vector layer (number of boreholes in the layer)
    info = g.parse_command('v.info',flags='t', map=inmap,quiet=True)
    n = int(info['areas'])

    # extrude each individual boreholes (one 3D layer for each borehole)
    for i in range(n-1):
         name = "bore"+str(ident[i]) # name of the 3D layer
         # selection/extraction of one element of the table
         g.run_command('v.extract',input=inmap, list=i+1, output='tmp',type='area', quiet=True, overwrite=True)
         # extrusion with base = limit of B formation (levels[i][1] = 850 for the first), height=thickness A-B (levelst[i][0])-float(levelst[i][1]) = 40 for the first)
         g.run_command('v.extrude', input='tmp', output=name, zshift=float(levels[i][1]),height= float(levelst[i][0])-float(levelst[i][1]),overwrite=True)
         g.run_command('g.remove', vect='tmp', quiet=True)

3) the resulting 3D layers (cylinders) in nviz (thickness of the interval A-B)

![](http://osgeo-org.1560.x6.nabble.com/attachment/4978801/3/boreholes.png)

4) it is not possible to create a single layer because v.patch writes always 2D layers. For example:
        
         # creation of a layer
         g.run_command('v.edit', tool='create', map=outmap)
       
         # extrude each borehole and add to the layer (v.patch)
         for i in range(n-1):
             g.run_command('v.extract',input=inmap, list=i+1, output='tmp',type='area', quiet=True, overwrite=True)
             g.run_command('v.extrude', input='tmp', output='tmp_extr', zshift=float(levels[i][1]),height= float(levelst[i][0])-float(levels[i][1]),overwrite=True)
             g.run_command('v.patch', flags='a', input='tmp_extr', output=outmap, quiet=True, overwrite=True)
             g.run_command('g.remove', vect='tmp,tmp_extr', quiet=True)

gives the result "WARNING: The output map is not 3D"

5) I repeat the process for the other intervals

![](http://osgeo-org.1560.x6.nabble.com/attachment/4978801/2/boreholes2.png)

6) result of interpolation of the surface (limit formations A-B) with r.surf.nnbathy (l algorithm)


![](http://osgeo-org.1560.x6.nabble.com/attachment/4978801/0/result.png)


Hoping that it may be useful to someone. The only (small) problem is you get lots of layers.
