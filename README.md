Inverse Kinematics Library
==========================

A lightweight implementation of the FABRIK solver.

[See the wiki page for details on how to use it](https://github.com/TheComet93/ik/wiki)

Overview
--------

One of the challenges was to design an interface  which  could  work  with any
scene graph/skeleton/animation system.  The  library  provides  a leightweight
interface for building a tree  and specifying positions and rotations for each
node.  The tree also holds information on effector targets and  chain  length.
The  tree is then preprocessed into a more optimal form for the  solver.  This
preprocessing step is necessary  whenever the tree is altered or effectors are
added or removed,  which,  in  practice,  should only occur once. Invoking the
solver  will  cause  a  series  of  computations  to  occur  on  the  internal
structures, the results of which will then be  mapped  back  onto the original
tree structure specified by the user. The user can then iterate  the  tree and
apply   the   solved   positions   and   rotations   back   to    his    scene
graph/skeleton/animation structure.

All  of the code was written in C89 and has no dependencies other than  the  C
standard  library.  Memory  debugging  facilities are in place to track memory
leaks.  On  linux,  backtraces can be generated to the respective malloc() and
free() calls.

Example usage
-------------

Here is a minimal working example that probably satisfies your needs.

```cpp
#include <ik/ik.h>

static void results_callback(struct ik_node_t* ikNode)
{
    /* Extract our scene graph node again */
    Node* node = (Node)ikNode->user_data;

    /* Apply results back to our engine's tree */
    node->SetWorldPosition(ikNode->position);
    node->SetWorldRotation(ikNode->rotation);
}

int main()
{
    /* Create a tree that splits into two arms */
    struct ik_node_t* root = ik_node_create(0);
    struct ik_node_t* child1 = ik_node_create(1);
    struct ik_node_t* child2 = ik_node_create(2);
    struct ik_node_t* child3 = ik_node_create(3);
    struct ik_node_t* child4 = ik_node_create(4);
    struct ik_node_t* child5 = ik_node_create(5);
    struct ik_node_t* child6 = ik_node_create(6);
    ik_node_add_child(root, child1);
    ik_node_add_child(child1, child2);
    ik_node_add_child(child2, child3);
    ik_node_add_child(child3, child4);
    ik_node_add_child(child2, child5);
    ik_node_add_child(child5, child6);

    /* Lets assume we are developing a game engine that has its own scene graph,
     * and lets assume it has the same structure as the tree created above.
     */
    Node* sceneRoot = GetSceneRoot();

    /* Store a pointer to each engine node to user_data so we can use it later */
    root->user_data = sceneRoot;
    child1->user_data = sceneRoot->GetChild(1);
    child2->user_data = sceneRoot->GetChild(2);
    child3->user_data = sceneRoot->GetChild(3);
    child4->user_data = sceneRoot->GetChild(4);
    child5->user_data = sceneRoot->GetChild(5);
    child6->user_data = sceneRoot->GetChild(6);

    /* Attach an effector on each arm */
    struct ik_effector_t* eff1 = ik_effector_create();
    struct ik_effector_t* eff2 = ik_effector_create();
    ik_node_attach_effector(child4, eff1);
    ik_node_attach_effector(child6, eff2);

    /* Each arm is composed of 3 nodes (2 segments), and we only want to control
     * that portion of the tree. */
    eff1->chain_length = 2;
    eff2->chain_length = 2;

    /* Create a solver and set up the results callback function, which gets
     * called once for every computed result. */
    struct ik_solver_t* solver = ik_solver_create(SOLVER_FABRIK);
    solver->apply_result = results_callback;

    /* We want to calculate rotations as well as positions */
    solver->flags |= SOLVER_CALCULATE_FINAL_ROTATIONS;

    /* Assign our tree to the solver and rebuild the data */
    ik_solver_set_tree(solver, root);
    ik_solver_rebuild_data(solver);
    ik_solver_solve(solver);
    solver->results_callback = results_callback;
    ik_solver_iterate_tree(solver);
}

```
