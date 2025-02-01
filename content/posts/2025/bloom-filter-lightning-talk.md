---
title: "OpenSCAD: More Satisfying than you Think!"
date: 2025-01-31
draft: false
summary: An introduction to OpenSCAD to encourage you to try it out
tags: ['Talking', 'OpenSCAD', 'CAD', '3D Printing']
---

A hardware engineer friend of mine recently introduced me to OpenSCAD. He used it in a production use-case, designing non-trivial plastic enclosures for electronic devices for 3D printing and then eventually plastic injection molding. This is not what this talk is about.

But it is where my journey into OpenSCAD began. In this presentation, I'm not going to teach you OpenSCAD, but rather try to inspire you to give it a go.

<iframe src="https://docs.google.com/presentation/d/e/2PACX-1vQh5BIVrUQe2OqnWmHGV698bwv1Lp2lU8wZ3_Ay7pH2jow0SxtHNSQEyj1mjOWqF5g5lMoouFBLnQfC/embed?start=false&loop=false&delayms=3000" frameborder="0" width="960" height="440" allowfullscreen="true" mozallowfullscreen="true" webkitallowfullscreen="true"></iframe>

If time allows, perhaps I'll give a demo based on one of earlier models.

{{<figure
    title="Wind-turbine model"
    src="/post-images/openscad/turbine.png"
    caption="Model that looks more complicated than it really is"
    align=center
>}}

Here is the code for a wind-turbine I made. Admittedly, it makes a prettier looking model than an effective wind turbine, but its a good model to step through as a demo.

To step through this, comment out the `full_model();` and uncomment the `blade();` lines at the end of the file; then use the `!` operator to work up through the `square` object to the `translate` transform, so you can see how blade is build. Remember that the `!` operator effectively executes just that part of the model and its dependencies. The `#` operator (which is not a comment) is also useful for highlighting that parts of the model, such as one of parts of a `difference` transform.

```openscad
$fn = 64;

outer_can_inside_d = 30;
outer_can_wall_w = 2;
outer_can_end_w = 1.5;
outer_can_h = 10;
blade_radial_w = 39;
tad = 0.1;
blade_thickness = 1.5;
blade_curve_radius = 52; // related to blade_radial_w
blade_twist_angle = 100;
blade_close_angle = 45; // how closed/shuttered/tucked in from perpendicular to tangent
blade_count = 6;


module blade() {
    translate([0,0,-blade_radial_w])
        linear_extrude(height=outer_can_h+2*blade_radial_w, twist=blade_twist_angle)
        translate([0,outer_can_inside_d/2 + tad,0])
        rotate([0,0,blade_close_angle])
        projection()
        translate([-blade_curve_radius - blade_thickness/2,0,0])
        rotate_extrude(angle=blade_curve_radius)
        translate([blade_curve_radius,0,0])
        square([blade_thickness,tad]);
}

module full_model() {
    difference() {
        union() {
            // central hub outer solid
            translate([0,0,-blade_radial_w])
                cylinder(h=outer_can_h + 2*blade_radial_w, d=outer_can_inside_d + outer_can_wall_w);
            // blades
            for(blade_i = [0:blade_count-1]) {
                rotate([0,0,blade_i * 360 / blade_count])
                    blade();
            }
        }
        // hub hollow
        translate([0,0,-blade_radial_w -tad])
            cylinder(h=outer_can_h + 2*blade_radial_w - outer_can_end_w, d=outer_can_inside_d);
        // rounding the bottom edges of the blades
        rotate([180,0,0])
            rotate_extrude(angle=360)
            translate([outer_can_inside_d/2,0,0]) {
                difference() {
                    square([blade_radial_w+1,blade_radial_w+1]);
                    circle(r=blade_radial_w);
                }
            }
        // rounding the top edges of the blades
        translate([0,0,outer_can_h+15])
            rotate_extrude(angle=360)
            translate([outer_can_inside_d/2,0,0]) {
                difference() {
                    square([blade_radial_w+1,blade_radial_w+1]);
                    circle(r=blade_radial_w);
                }
            }
        
    }
}

full_model();
//blade();
```
