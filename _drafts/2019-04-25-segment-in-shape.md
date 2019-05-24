---
layout: post
title: "Finding if a line segment is inside a polygon"
date: 2019-04-25
author: yubbit
categories: ['Mathematics', 'Computational Geometry']
---

I think I finally found a way to determine if a line segment is inside a given simple polygon. Diagrams, code, and explanations to come.

1. If both of its endpoints are outside the polygon, then it is not inside the polygon
2. If one of its endpoints is inside the polygon, then it is inside the polygon
3. If both of its endpoints are inside the polygon, then it is inside the polygon
4. If both of its endpoints are on either a vertex or edge of the polygon:
   1. Build a set S of all line segments. At the intersection points, divide the polygon's line segment into two segments, then add those into S.
   2. Begin at an intersection point
       1. Travel along the line segment being evaluated
       2. If another intersection point is reached, follow either of the two possible segments
       3. Follow the line segments until you reach the starting intersection point
       4. Repeat this, except at the second intersection, follow the other line segment
       5. You now have two polygons. Add their areas, calculated using the shoelace algorithm. If their areas are greater than that of the original shape, then the line segment is outside the polygon, otherwise, it is inside
