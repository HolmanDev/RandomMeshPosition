# Sample Random Position on a Mesh Within a Radius
<img src="https://github.com/user-attachments/assets/85f625c6-4c93-422f-9985-c4b821ef8fc4" width="300px"/>

This article explores an algorithm designed to select a uniformly random position on the surface of a mesh within a sphere of specified radius. 
The algorithm incorporates principles of vector math, geometry, and linear algebra. Let's dive into the key ideas behind its implementation.

## Calculating Triangle Areas with the Cross Product

To sample a random position, the algorithm first identifies triangles within the mesh that intersect the sphere. 
It then selects a triangle based on the probability proportional to its area. To calculate each triangle's area, the **cross product** is used:

$\text{Area} = \frac{1}{2} \|\vec{v}_2 - \vec{v}_1 \times \vec{v}_3 - \vec{v}_1\|$

Here, $\vec{v}_1$, $\vec{v}_2$, and $\vec{v}_3$ are the vertices of the triangle. 
The cross product yields a vector orthogonal to the plane of the triangle, whose magnitude (by definition) corresponds to twice the triangle's area.

*Wikipedia*: https://en.wikipedia.org/wiki/Cross_product

## Line and Plane Projections

Determining whether a triangle lies within the sphere involves checking distances between the sphere's center and:

1. The triangle's edges (line segments).
2. The plane of the triangle.

### Distance to Line Segments
For each edge, the algorithm orthogonally projects the center of the sphere onto the line defined by the edge. 
It clamps this projection to the segment bounds, yielding the closest point. The distance between this point and the sphere's center determines whether the edge intersects the sphere.

*Wikipedia*: https://en.wikipedia.org/wiki/Vector_projection

### Distance to the Triangle Plane
For cases where no edge intersects the sphere, the algorithm orthogonally projects the center onto the triangle's plane. 
This requires finding the point on the plane closest to the sphere's center and verifying if it lies within the triangle.

## Change of Basis for Plane Projections

Once the sphere's center is projected onto the triangle's plane, we need to express this projected point in terms of the triangle's basis vectors. A **change of basis** achieves this:

1. The triangle is represented by two sides, $\vec{s}_A = \vec{v}_2 - \vec{v}_1\) and \(\vec{s}_B = \vec{v}_3 - \vec{v}_1$, and their normal vector $\vec{n} = \vec{s}_A \times \vec{s}_B$.
2. A transformation matrix $\mathbf{B}$ is constructed from these vectors.
3. The projected point $\vec{P}$ is transformed into barycentric coordinates relative to the triangle using $\mathbf{B}^{-1} \vec{P}$.

*Wikipedia*: https://en.wikipedia.org/wiki/Change_of_basis

## Inside-Triangle Check

After transforming the projected point, its barycentric coordinates $(a, b)$ are verified. A point lies within the triangle if:

$0 \leq a \leq 1, \quad 0 \leq b \leq 1, \quad \text{and } a + b \leq 1$

This condition ensures the point is bounded by the triangle's edges.

## Algorithm Workflow

1. **Triangle Filtering**:
   - For each triangle, calculate distances from the sphere's center to edges and the plane.
   - Filter out triangles entirely outside the sphere.

2. **Weighted Sampling**:
   - Use the cross product to calculate areas of the remaining triangles.
   - Select a triangle with probability proportional to its area.

3. **Random Point Selection**:
   - Sample a random barycentric coordinate pair $(a, b)$ satisfying $0 \leq a, b, a+b \leq 1$.
   - Use these coordinates to compute a point on the triangle.

4. **Validation**:
   - Verify the point lies within the sphere in 3D space; if not, retry from step 2.
