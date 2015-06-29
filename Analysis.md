
* [less](http://lesscss.org/): Less is a CSS pre-processor, meaning that it extends the CSS language, adding features that allow variables, mixins, functions and many other techniques that allow you to make CSS that is more maintainable, themable and extendable.


Start with `Build.PL`. Aim for structure not details.

`Build.PL` only deals with dependencies.

Folders( use `tree -dL 1` to have this):

➜  Slic3r git:(master) ✗ tree -dL 1
.
├── lib: the lib `./slic3r.pl` relies on.
├── t: for running tests. See the module [page](http://perldoc.perl.org/Test/Simple.html)
├── utils: some utils scripts, for post-processing or something
├── var: this is picture assets folder.
└── xs: [perlxs](http://perldoc.perl.org/perlxs.html)

➜  xs git:(master) ✗ tree -L 1
.
├── blib
├── _build: config_dir
├── Build: use it with `perl Build`
├── Build.PL: generates `Build`
├── buildtmp
├── lib: 
├── MANIFEST [manifest](http://perldoc.perl.org/ExtUtils/Manifest.html)
├── MANIFEST.SKIP
├── MYMETA.json
├── MYMETA.yml
├── src
├── t
└── xsp: some definitions, C/C++?

This [tutorial](http://perldoc.perl.org/ExtUtils/MakeMaker/Tutorial.html) explains the structure of `xs`.

for `@{}`, [perlref](http://perldoc.perl.org/perlref.html)

echo "source ~/perl5/perlbrew/etc/bashrc" | tee -a ~/.zshrc

cpanm WithXSpp

config_dir => '_build',
    orig_dir => $orig_dir, ------>'orig_dir' => '/home/hustlion/Slic3r/xs',


➜  _build git:(master) ✗ tree
.
├── auto_features: empty
├── build_params
├── cleanup: specify how to clean files
├── config_data: empty
├── features: empty
├── magicnum: some number generated?
├── notes: empty
├── prereqs: empty
└── runtime_params: 'install_base' => '/home/hustlion/perl5'

'dist_version_from' => 'lib/Slic3r/XS.pm',

lib/Slic3r/XS.pm specifies many packages.

'extra_xs_dirs' => [
      '.',
      'xsp'
    ],

'libdoc_dirs' => [
      'blib/lib',
      'blib/arch'
    ]

'metafile' => 'META.yml',
    'metafile2' => 'META.json',
    'module_name' => 'Slic3r::XS'
'mymetafile' => 'MYMETA.yml',
    'mymetafile2' => 'MYMETA.json',


is C files compiled to Perl? Or,, they run together?

If an XSUB name contains ::, it is considered to be a C++ method.
The generated Perl function will assume that its first argument is an object pointer

So,,, perl function?

There gottta be something to tell slic3r that XS folder is there. `/Build.PL`, line 146, `if (!$gui) {`, there it is.


In Build.PL


haha, got it

`perl Build distclean`
and the xs:
➜  xs git:(master) ✗ tree -L 1
.
├── Build.PL
├── lib
├── MANIFEST
├── MANIFEST.SKIP
├── src
├── t
└── xsp

4 directories, 3 files

`perl Build.PL --xs` actually build the xs alone :smile: wow that's it!
src, t, xsp, lib, Build.PL, MANIFEST, MANIFEST.SKIP should be written. Others are generated.

`Installing /home/hustlion/perl5/lib/perl5/x86_64-linux/Slic3r/XS.pm`

So these libs are installed to the system! This is system wide. You can copy one test file from the `t` folder. And run it with `perl 03_point.t`. The test is passed!

`perldoc FindBin` great!

In the test, `use Slic3r::XS;` is used. This is importing the system package. How about `slic3r.pl`?

/home/hustlion/perl5/lib/perl5/x86_64-linux/auto/Slic3r/XS/XS.so is the actual lib! `.so` for C/C++! like `.exe`! So the performance is ensured! The built one is here: `/home/hustlion/Slic3r/xs/blib/arch/auto/Slic3r/XS`. Copied to!

/home/hustlion/perl5/lib/perl5/x86_64-linux/Slic3r/XS.pm is the interface! Copied from the lib folder as it is!

blib is shorthand for built libraries, right? just copy `arch` (for archytech..) as the executable and `lib` (for the library) as interface!

So `Build.PL` not only checks the dependencies and also does a important job - install the XS to system! Install it by this line

```` python
my $res = system $cpanm, @cpanm_args, '--reinstall', '--verbose', './xs';
````

interpret as:
cpanm --reinstall --verbose ./xs

so if comiple and install manually:
cd Slic3r
./xs/Build distclean
cpanm --reinstall --verbose ./xs

So the C++ source files are provided in ./xs/src. And it's compiled and installed by cpanm to the system.

So, we should analyse XS.pm and find out the mapping between modules and c++ files, then we know what to modify to encode our algorithm!

/home/hustlion/Slic3r/xs/lib/Slic3r/XS.pm, line 71, `package Slic3r::ExtrusionPath::Collection;`

X:\Slic3r-master\lib\Slic3r\Fill\Honeycomb.pm, line 98 to 155.

# connect paths
if (@paths) { # prevent calling leftmost_point() on empty collections
my $collection = Slic3r::Polyline::Collection->new(@paths);
@paths = ();
foreach my $path (@{$collection->chained_path_from($collection->leftmost_point, 0)}) {
if (@paths) {
# distance between first point of this path and last point of last path
my $distance = $paths[-1]->last_point->distance_to($path->first_point);

if ($distance <= $m->{hex_width}) {
$paths[-1]->append_polyline($path);
next;
}
}

# make a clone before $collection goes out of scope
push @paths, $path->clone;
}



before reading src, do `./xs/Build distclean`

got it finally.

/home/hustlion/Slic3r/xs/src/libslic3r/PolylineCollection.cpp
void chained_path_from(Point start_near, PolylineCollection* retval, bool no_reverse = false) const;

haha.

The algorithm is:
```` c++
while (!my_paths.empty()) {
        // find nearest point
        int start_index = start_near.nearest_point_index(endpoints);
        int path_index = start_index/2;
        if (start_index % 2 && !no_reverse) {
            my_paths.at(path_index).reverse();
        }
        retval->polylines.push_back(my_paths.at(path_index));
        my_paths.erase(my_paths.begin() + path_index);
        endpoints.erase(endpoints.begin() + 2*path_index, endpoints.begin() + 2*path_index + 2);
        start_near = retval->polylines.back().last_point();
    }
````
/home/hustlion/Slic3r/xs/src/libslic3r/Point.hpp
int nearest_point_index(const Points &points) const;

````c++
int
Point::nearest_point_index(const Points &points) const
{
    PointConstPtrs p;
    p.reserve(points.size());
    for (Points::const_iterator it = points.begin(); it != points.end(); ++it)
        p.push_back(&*it);
    return this->nearest_point_index(p);
}
````
```` C++
int
Point::nearest_point_index(const PointConstPtrs &points) const
{
    int idx = -1;
    double distance = -1;  // double because long is limited to 2147483647 on some platforms and it's not enough
    
    for (PointConstPtrs::const_iterator it = points.begin(); it != points.end(); ++it) {
        /* If the X distance of the candidate is > than the total distance of the
           best previous candidate, we know we don't want it */
        double d = pow(this->x - (*it)->x, 2);
        if (distance != -1 && d > distance) continue;
        
        /* If the Y distance of the candidate is > than the total distance of the
           best previous candidate, we know we don't want it */
        d += pow(this->y - (*it)->y, 2);
        if (distance != -1 && d > distance) continue;
        
        idx = it - points.begin();
        distance = d;
        
        if (distance < EPSILON) break;
    }
    
    return idx;
}
````

I remember I had analysed this before. I was just not sure how this will work. :smile: now I understand.

And, this function just choose the shortest one based on these point. Given the same points, the best candidate is certain (when there is too many, maybe not). Need to understand how these points are generated. And then decide where is the part for optimization.

Polygon partion->connect polygon (polygoncollection?)->connect point.

The second part maybe.

/home/hustlion/Slic3r/xs/src/polypartition.h with comments.

/home/hustlion/Slic3r/xs/src/libslic3r/MotionPlanner.cpp :
islands, environment, this is it!
// Now check whether points are inside the environment.
    Point inner_from    = from;
    Point inner_to      = to;
    bool from_is_inside, to_is_inside;
    
    if (!(from_is_inside = env.contains(from))) {
        // Find the closest inner point to start from.
        inner_from = this->nearest_env_point(env, from, to);
    }
    if (!(to_is_inside = env.contains(to))) {
        // Find the closest inner point to start from.
        inner_to = this->nearest_env_point(env, to, inner_from);
    }
// perform actual path search
    MotionPlannerGraph* graph = this->init_graph(island_idx);
    Polyline polyline = graph->shortest_path(graph->find_node(inner_from), graph->find_node(inner_to));
    
    polyline.points.insert(polyline.points.begin(), from);
    polyline.points.push_back(to);

std::vector<MotionPlannerGraph*> graphs;
class MotionPlannerGraph;

start from /home/hustlion/Slic3r/xs/src/libslic3r/MotionPlanner.cpp line 235

line 319, shortest path: 
`MotionPlannerGraph::shortest_path(size_t from, size_t to)`

Voronoi-generated edges

likely dijkstra? This is deep now. Need to sort the workflow out before making conclusion.

# xs routine
perl Build.PL --xs



