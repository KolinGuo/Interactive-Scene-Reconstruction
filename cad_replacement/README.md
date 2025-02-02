# CAD Replacement

Given reconstructed mesh segments of layouts (e.g., wall, floor) and object instances in a scene, we estimate the supporting relations and construct a contact graph, and replace object meshes with CAD models accounting for predicted object semantics, geometric similarity, and physical constriants imposed by the contact graph. 

The two packages under this directory:

- **map_proc** : contains the major implementations of contact graph construction and CAD replacement. See `map_proc/launch/map_processing.launch` for options and `map_proc/src/map_processing_node.cpp` for detailed implementations.

- [**pg_map_ros**](pg_map_ros/pg_map_ros/) : a ros package that defines a class of graph structures (parse graph) and provides functions to construct, save, load and visualize a parse graph.

*Note: Before running the following steps, please make sure you have properly install dependencies and build the packages following [this instruction](../assets/INSTALL.md).*


## 1. Usage

### 1.0 Please recompile to update changes

``` shell
catkin build -DPYTHON_EXECUTABLE=/usr/bin/python3 panoptic_mapping_pipeline map_proc scene_builder gazebo_simulation
```

If you need debugging tags, do
``` shell
catkin build -DPYTHON_EXECUTABLE=/usr/bin/python3 -DCMAKE_BUILD_TYPE=Debug -DCMAKE_CXX_FLAGS_DEBUG="-g -fsanitize=address" panoptic_mapping_pipeline map_proc scene_builder gazebo_simulation
```

### 1.1 Download CAD dataset

Unzip "cad_dataset_sapien.zip" into repo's root directory.

### 1.2 Launch the `map_processing_node`

First make sure you have the result of panoptic mapping under `Interactive-Scene-Reconstruction/output/$SEQUENCE/`, where `$SEQUENCE` is the sequence name specified when launching the panoptic mapping pipeline (`sceneNN_test` by default). Should already be in repo as `mobile_pnp_0-1` and `mobile_pnp_noisy_0-1`.

Then launch the ros node via:

``` shell
roslaunch map_proc mobile_pnp_map_processing.launch sequence:=mobile_pnp_0-1
```

*Note that to close a viewer and move on to the next step, press `a` on your keyboard*.

Please refer to `map_proc/launch/map_processing.launch` for more running options. The generated contact graph is saved under `Interactive-Scene-Reconstruction/output/$SEQUENCE/contact_graph/`. While `contact_graph_seg.json` stores the initialized contact graph without CAD replacement, `contact_graph_cad.json` stores the final output.

### 1.3 Visualize contact graph

We also provide a tool to visualize contact graphs. In the root folder (`Interactive-Scene-Reconstruction/`), run:

``` shell
conda activate robot-scene-recon
python cad_replacement/pg_map_ros/pg_map_ros/pg_viewer/launch_viewer.py -c output/sceneNN_test/contact_graph/contact_graph_cad.json
```


## 2. Technical Explainations

We summarize our pipeline as below; note that it yields some improvements based on the description in the paper.

```c++
//// 1. Load map object segments
std::unordered_map<std::string, std::vector<ObjCAD::Ptr>> cad_database;
LoadObjectDatabase (cad_database);

//// 2. Load map object segments
std::vector<Obj3D::Ptr> objects;
LoadInstanceSegments(cad_database, objects);

//// 3. Estimate supporting relations and initialize contact graph
// Decide supporting parents for all objects
std::unordered_map<Obj3D::Ptr, std::unordered_map<Obj3D::Ptr, Eigen::Vector4f>> parent_child_map;
std::queue<Obj3D::Ptr> obj_to_check; // Nodes connected to the root node in the contact graph
DecideSupportingParents (objects, parent_child_map, obj_to_check);

// Refine objects using the supporting planes
UpdateObjectsViaSupportingRelations (objects, parent_child_map, obj_to_check);

// Bottom-up build the contact graph
pgm::ConceptNode root (0, "Room");
pgm::ParseGraphMap::Ptr contact_graph (new pgm::ParseGraphMap(root));
BuildContactGraph (parent_child_map, gt_objects, obj_to_check, contact_graph);

//// 4. Coarse scoring-based CAD matching
std::unordered_map<Obj3D::Ptr, std::vector<ObjCADCandidate::Ptr>> cad_candidates_map;
MatchCADToSegmentsCoarse (objects, cad_database, cad_candidates_map);

//// 5. Prune CAD candidates by checking physical constraints, and return k best candidates for each map object
int k_max = 40;
std::unordered_map<Obj3D::Ptr, std::vector<ObjCADCandidate::Ptr>> cad_selected_candidates;
for (auto& cad_candidates_pair: cad_candidates_map)
{    
    int count = 0; 
    std::vector<ObjCADCandidate::Ptr> candidate_vec;
    for (auto& candidate: cad_candidates_pair.second)
    {           
        // Check supported area
        if (!ValidateSupportedArea (candidate, 0.3))
            continue;
        // Check collision & supporting area when supporting other objects
        else if (ValidateSupportingAffordance (candidate))
        {
            candidate_vec.push_back(candidate);
            count ++;                
        }

        if (count > k_max)
            break;
    }
    cad_selected_candidates.insert(std::make_pair(cad_candidates_pair.first, candidate_vec));
}

//// 6. Fine optimization-based CAD alignment and matching
// Compute possible collisions
std::unordered_map<Obj3D::Ptr, std::unordered_map<Obj3D::Ptr, float>> possible_collision_map;
ComputePotentialCollisionPairs (cad_selected_candidates, possible_collision_map);
// Align CAD models
MatchCADToSegmentsFine (objects, cad_selected_candidates);

//// 7. Global collision_free regulation
std::unordered_map<Obj3D::Ptr, ObjCADCandidate::Ptr> map_candidate;
GlobalRegulation (objects, cad_selected_candidates, possible_collision_map, map_candidate);

//// 8. The final contact graph
FillContactGraph (objects, gt_objects, map_candidate, contact_graph);
```


## 3. Known Limitations

We propose a complex pipeline consisting of tridational methods to replace reconstructed object meshes with CAD models, while maintaining the physical plausibility of the scene. Some known limitations of the pipeline are:

- **Some of the replaced CAD models may be in wrong orientations, and the selected CAD models may not match well with novel map objects**, as the algorithm relying on simple geometric features (e.g., planes, 3D bounding boxes) may be potentially fragile when the reconstructed meshes are noisy and incomplete. However, we are aware that the orientation estimation can be handled robustly using deep learning-based methods.

- **Supporting by wall and hanging may not be well-handled**, while in fact we are not aware of any other works that can perfectly deal with these complex reelations at the scene-level.

- **The pipeline takes seconds to minutes to run for a single scene** as we loop over the whole CAD database in some of the steps, and use dense collision detection and point cloud/mesh processing to check for the physical plausibility.

