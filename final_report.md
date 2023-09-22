# Curvature Adaptive Remeshing
### Ivan Pađen - the final report for Google Summer of Code 2023 with the CGAL Project

# 1. Background

<img align="right" width="460px" src="https://raw.githubusercontent.com/ipadjen/gsoc2023/main/figs/david_comparison.png"/>

Surface remeshing improves mesh quality, optimizes data storage, and enhances visualization. This is beneficial for applications such as numerical simulations and animation. The goal of this project was to implement the curvature-adaptive surface remeshing by [Dunyach et al. (2013)](https://diglib.eg.org/handle/10.2312/conf.EG2013.short.029-032) into the CGAL's isotropic remeshing framework. The method operates iteratively, performing the following steps:
* Edge collapsing and splitting based on targeted resolution,
* Edge flipping to improve valence (target is 6 in the interior and 4 on boundary),
* Tangential smoothing to optimize vertex distribution, and
* Vertex reprojection onto the original surface to preserve exactness.

The main contributions of this project were made in edge collapsing and splitting, and the tangential smoothing. The new edge splitting and collapsing rules depend on the local target edge length which is calculated from the vertex sizing field. The sizing field is derived from the discrete curvature implemented in [a previous GSoC's contribution](https://summerofcode.withgoogle.com/programs/2022/projects/rqwWjZy3). This means that regions with a larger absolute curvature will result in smaller triangles and vice-versa.

The new tangential smoothing was implemented to account for varying edge lengths. Weights of the tangential smoothing are proportional to the triangle area divided by the sizing field at triangle's barycenter.

The implementation is independent of a data structure and it works on whole meshes, as well as selected surfaces. It also works with feature preservation and constraints. Finally, the functionality was incorporated in GUI through the polygon mesh processing plugin of the CGAL's Polyhedron demo.

# 2. Work completed
In summary, my contributions to the project include the following:

* Introduction of the `PMPSizingField` concept,
* Implementation of the `Adaptive_sizing_field` class,
* Introduction of the `tangential_relaxation_with_sizing()` function that takes sizing field values as weights in tangential relaxation,
* Changes to `split_long_edges()` and `constraints_are_short_enough()` functions to work with the sizing field,
* Implementation of changes to Polyhedron demo (`Isotropic_remeshing_plugin.cpp` and the GUI),
* Documentation, example `isotropic_remeshing_with_sizing.cpp`, and a test `remeshing_quality_test.cpp`.

### 2.1  The new PMPSizingField concept
The starting point for the project was the initial work done in the original [pull request](https://github.com/CGAL/cgal/pull/4891), where the `target_edge_length` argument was replaced by a `Uniform_sizing_field` class derived from a `Sizing_field_base` virtual class. I generalized the `Sizing_field_base` and `Uniform_sizing_field` classes with a vertex point map template argument. Originally, the functions used the default vertex point map of a polygon mesh; this is now a default template argument.

From the `Sizing_field_base`, I created the new [PMPSizingField](https://cgal.github.io/4891/v1/Polygon_mesh_processing/classPMPSizingField.html) concept that can be used to create new sizing fields. The classes created from the `PMPSizingField` concept should contain mesh edge splitting and collapsing rules through `is_too_long()` and `is_too_short()` functions, as well as a function to define a new point for edge splitting `split_placement()`.

Other functions that initially used prescribed target edge lengths were modified to accomodate the concept. This includes `split_long_edges()` and `constraints_are_short_enough()` functions.

### 2.2 Adaptive sizing field

I impemented a new adaptive sizing field class in `Adaptive_sizing_field.h`. It uses the approach proposed by [Dunyach et al (2013)](https://diglib.eg.org/handle/10.2312/conf.EG2013.short.029-032). Similar to the `Uniform_sizing_field`, edges are split if they are longer than the 4/3 of the target edge length and collapsed if they are shorter than the 4/5 of the target edge length. The target edge length is local and calculated from a discrete curvature.

We start by calculating a discrete curvature using the `interpolated_corrected_principal_curvatures()` implemented in a [GSoC 2022 project](https://summerofcode.withgoogle.com/programs/2022/projects/rqwWjZy3). The sizing for individual vertices is calculated as

```math
L\left( \mathbf{x}_i \right) = \sqrt{6 \epsilon / \mathbf{\kappa}_i - 3 \epsilon^2},
```

where the curvature $\kappa$ is defined as $\kappa_i = \mathrm{max} \left( \kappa_{i_\mathrm{max}},\kappa_{i_\mathrm{min}} \right)$, and $\epsilon$ is a user-defined error tolerance. Finally, given a two endpoints of an edge, the local target edge length is calculated as

```math
L\left(e\right) = \mathrm{min} \left( L\left( \mathbf{x}_1 \right), L\left( \mathbf{x}_2 \right) \right)
```

Sizing values for vertices  $L\left( \mathbf{x}_i \right)$ are stored as a vertex property map. The (target) edge sizing $L\left(e\right)$ is calculated on the fly when the edge length is checked for splitting or collapsing with `is_too_long()` and `is_too_short()` functions, respectively. Furthermore, to prevent extreme values, the edge sizing is constrained between user-defined minimum and maximum limits.

The sizing field can be calculated for the whole mesh or a selection. In case of a selection, the curvature is calculated from the selection expanded with one layer of neighbouring faces (`expand_face_selection()` function). Additionally, as derived from the `Sizing_field_base`, the input can be a user-defined vertex point map.

The curvature calculation also incorporates the `ball_radius` named parameter, which can smooth a noisy curvature field

The sizing field is updated when the edge is split. Since the edge is split at the midpoint, the new vertex sizing is calculated as an average of the two initial sizing values.

### 2.3 Tangential relaxation
I implemented a new tangential relaxation function `tangential_relaxation_with_sizing()` which moves points using the expression

```math
\mathbf{c}_i=\frac{\sum_{j \in \mathcal{T}_i}\left|t_j\right| L\left(\mathbf{b}_{j}\right)\mathbf{b}_{j}}{\sum_{j\in\mathcal{T}_{i}}\left|t_{j}\right| L\left(\mathbf{b}_{j}\right)},
```

where $\left|t_j\right|$ is an area of an incident triangle and $L\left(\mathbf{b}_j\right)$ the sizing field at the barycenter $\mathbf{b}_j$ of the incident triangle, calculated as an average of triangle sizing values. The rest of the implementation, such as the rules when the point relocation is large and could cause degenerate triangles, is identical to the existing implementation in CGAL.

### 2.4 Polyhedron demo
I updated the `Isotropic_remeshing_plugin.cpp` of the Polyhedron demo with the adaptive sizing field, as well as added the adaptive sizing field functionalities to the GUI with the Qt Designer.

### 2.5 Example, test, and documentation
I updated the `isotropic_remeshing_with_example.cpp` test to showcase the new functionalities.

I added the `remeshing_quality_test.cpp` that calculates the triangle quality value as $Q(t) = \frac{6}{\sqrt{3}}\frac{A_t}{S_t E_t}$, where $A_t$ is the triangle area, $S_t$ the half perimeter, and $E_t$ the longest edge length. The $Q$ value equals 1 for equilateral triangles, and lowers as the difference from an equilateral triangle increases.
The test calculates average and minimum quality $\overline{Q}$, $Q_{\mathrm{min}}$, as well as the minimum angle in the mesh $\theta_{\mathrm{min}}$ and the average of minimum angles in all triangles $\overline{\theta_\mathrm{min}}$.

Lastly, I updated the reference and user manual to reflect on all of my developments.


# 3. Future work
[Khan et al. (2020)](https://par.nsf.gov/servlets/purl/10293572) made a comprehensive literature review on surface remeshing algorithms. Future work could explore other remeshing options and enhancements highlighted in the literature, especially those promising improvements in robustness and resulting mesh quality.

# 4. Conclusion
It was a fun and useful project that combined different skills such as computational geometry, C++ template metaprogramming, and Qt. I would like to thank my mentors Jane Tournois and Sebastien Loriot for insightful discussions and consistent support.
