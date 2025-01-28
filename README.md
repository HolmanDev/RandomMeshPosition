# Random mesh position within a sphere
<img src="https://github.com/user-attachments/assets/85f625c6-4c93-422f-9985-c4b821ef8fc4" width="300px"/>

Select a uniformly random position on a mesh within a radius from a point.

Written for the game engine Unity in C#, but the concepts are easily transferred to other fields.

The code was originally developed to select a random walking destination for a Navmesh agent.

## Code snippets
*This is the entire code*

Required namespaces.
```C#
using UnityEngine;
using UnityEngine.AI;
using System.Collections;
using System.Collections.Generic;
```

A simple triangle struct to organize the vertex data.
```C#
// Struct that holds three Vector3's
struct Triangle
{
    public Vector3 v1;
    public Vector3 v2;
    public Vector3 v3;

    public Triangle(Vector3 v1, Vector3 v2, Vector3 v3)
    {
        this.v1 = v1;
        this.v2 = v2;
        this.v3 = v3;
    }
}
```

**Main function**: Returns a random position within a sphere, on the mesh specified by its vertices and indices (triangles). Indices are excpected to be grouped in pairs of three, one group for each triangle. This is why the for-loop looks at three indices (and vertices) at a time.
```C#
/// <summary>
/// Get random XYZ coordinate on a mesh within a sphere with specified radius and origin.
/// </summary>
public Vector3 RandomPosOnMeshInRadius(Vector3[] vertices, Vector3[] indices, Vector3 center, float radius)
{
    List<Triangle> inRangeTriangles = new List<Triangle>();
    float sqrRadius = radius * radius;
    for (int i = 0; i < indices.Length; i += 3)
    {
        int i1 = indices[i];
        int i2 = indices[i + 1];
        int i3 = indices[i + 2];
        Vector3 v1 = vertices[i1];
        Vector3 v2 = vertices[i2];
        Vector3 v3 = vertices[i3];

        float distanceTo12 = DistanceToLineSegment(center, v1, v2);
        float distanceTo23 = DistanceToLineSegment(center, v2, v3);
        float distanceTo31 = DistanceToLineSegment(center, v3, v1);
        bool anyEdgeInside = distanceTo12 < radius || distanceTo23 < radius || distanceTo31 < radius;
        // If any edge is inside the radius, or if the center is within such a big triangle that no edges are within the radius
        if(anyEdgeInside || DistanceToProjectedPointOnTriangle(v1, v2, v3, center) < radius)
        {
            inRangeTriangles.Add(new Triangle(v1, v2, v3));
        }
    }

    // Calculate triangle areas
    List<float> areas = new List<float>();
    float totalArea = 0.0f;
    foreach(Triangle tri in inRangeTriangles)
    {
        Vector3 v1 = tri.v1;
        Vector3 v2 = tri.v2;
        Vector3 v3 = tri.v3;
        float area = Vector3.Cross(v2 - v1, v3 - v1).magnitude / 2.0f; // https://en.wikipedia.org/wiki/Cross_product
        areas.Add(area);
        totalArea += area;
    }

    // Loop until a point inside the radius is found. Stop at an unreasonable amount of attempts.
    int maxAttempts = 100;
    int attempts = 0;
    while (attempts < maxAttempts)
    {
        //Select a weighted random triangle based on triangle areas
        float randomFloat = Random.Range(0.0f, totalArea);
        float sum = 0.0f;
        int selectedTriangleIndex = -1;
        foreach (float area in areas)
        {
            selectedTriangleIndex++;
            sum += area;
            if (sum > randomFloat) break;
        }
        // If no suitable triangle is found, return the center
        if (selectedTriangleIndex == -1) return center;
        // If a suitible triangle was found, store it as the "selected" triangle
        Triangle selectedTriangle = inRangeTriangles[selectedTriangleIndex];

        // On the selected triangle, try to select a random point
        Vector3 v1 = selectedTriangle.v1;
        Vector3 v2 = selectedTriangle.v2;
        Vector3 v3 = selectedTriangle.v3;
        float x = Random.Range(0.0f, 1.0f);
        float y = Random.Range(0.0f, 1.0f - x);
        Vector3 randomPosition = v1 + x * (v2 - v1) + y * (v3 - v1);
        if ((randomPosition - center).sqrMagnitude < sqrRadius)
        {
            return randomPosition;
        }
        attempts++;
    }
    return center; // Return the center position if no point is found
}
```

**Helper function 1**: Calculates the distance from a point in 3D space to a line segment defined by the end points A and B.
This takes care of triangles who have at least one edge inside the sphere's radius, which is the most common case.
```C#
// Calculate the distance from a point to a line segment.
private float DistanceToLineSegment(Vector3 point, Vector3 A, Vector3 B)
{
    Vector3 AB = B - A;
    Vector3 AP = point - A;
    float abLength = AB.magnitude;
    
    // Project the point onto the line defined by the segment
    float t = Mathf.Clamp(Vector3.Dot(AP, AB) / (abLength * abLength), 0f, 1f);
    Vector3 closestPoint = A + t * AB;

    // Calculate the distance from the point to the closest point on the segment
    return Vector3.Distance(point, closestPoint);
}
```

**Helper function 2**: Returns the distance to the closest point on a triangle's plane, but only if that point is inside the triangle. If not, float.MaxValue is returned instead.
This takes care of the special case where the center of the sphere is inside of triangle so large that no edges intersect it.
```C#
// Check if the closest point on the triangle's plane is within the triangle's bounds.
// If it is, return the distance. If not, return infinity.
private float DistanceToProjectedPointOnTriangle(Vector3 v1, Vector3 v2, Vector3 v3, Vector3 point)
{
    Vector3 sideA = v2 - v1;
    Vector3 sideB = v3 - v1;
    Vector3 triangleNormal = Vector3.Cross(sideA, sideB);
    Vector3 P = Vector3.ProjectOnPlane(point, triangleNormal); // Point on plane through origin

    // Change of basis
    Matrix4x4 triangleBase = new Matrix4x4();
    triangleBase.SetColumn(0, sideA);
    triangleBase.SetColumn(1, sideB);
    triangleBase.SetColumn(2, triangleNormal);
    triangleBase.SetColumn(3, new Vector4(0, 0, 0, 1));
    // Bx' = x => x' = inv(B)x
    Vector3 P_based = triangleBase.inverse * P;

    // Check if the point is inside the triangle
    float a = P_based.x;
    float b = P_based.y;
    if(a >= 0.0f && a <= 1.0f && b >= 0.0f && b <= 1.0f-a)
    {
        Vector3 trianglePlaneOffset = Vector3.Project(v1, triangleNormal);
        Vector3 worldspaceP = P + trianglePlaneOffset;
        return Vector3.Distance(worldspaceP, point);
    }
        
    return float.MaxValue;
}
```
