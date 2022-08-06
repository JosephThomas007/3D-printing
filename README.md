# 3D printing of Crown Gear meshed with Cosine Spur pinion using IceSL software. 


--CROWN GEAR MESHING A COSINE PINION <br />
-- Reference article to get cosine pinion equations: The generation principle and mathematical models of a novel cosine gear drive, Shanming Luo, 14, February, 2008

																	--FUNCTIONS--

--Function-1: cosine_pinion_maker<br />
----This generates cosine pinion gear.<br />
--1.coordinates of cosine pinion in YZ-plane-->extruded in X-direction.<br />
--2.translated by in X-direction to achieve meshing position.<br />
--3.create a hole at the centre of cosine_pinion.<br />
```

function cosine_pinion_maker(m_n,z_pinion,b,crown_d_p)
	local cosine_pinion_points={}
	local dir_x=v(b+2, 0, 0)
	local index=1
	for n=0,(z_pinion), 0.02
	do
		theta=((2*math.pi)/z_pinion)*n
		y_cosine=((m_n*z_pinion/2)+(h_a*math.cos(z_pinion*theta)))*math.sin(theta) -- Reference article equation No: 8
		z_cosine=((m_n*z_pinion/2)+(h_a*math.cos(z_pinion*theta)))*math.cos(theta) -- Reference article equation No: 8
		cosine_pinion_points[index]=v(0,y_cosine,z_cosine ) --2D proile of cosine
		index=index+1
	end
	local cosine_pinion=linear_extrude(dir_x, cosine_pinion_points) -->Step-1.
	cosine_pinion=translate(crown_d_p/2-2,0,0)*cosine_pinion  -->step-2.
	index=1
	hole=translate(crown_d_p/2-2,0,0)*rotate(90,Y)*cylinder(10,b+2) -->step-3.
	cosine_pinion=difference(cosine_pinion,hole) -->step-3.
	return cosine_pinion

end

```

--Function-2: crown_workpiece<br />
----This generates complete workpiece for crown gear.<br />
--1.2D profile of workpiece--->revolved around Z-axis by 360 degrees.<br />
--2.translated in Z-direction to mesh with cosine pinion.<br />
```

function crown_workpiece(b,h_a,crown_d_p,cosine_d_p)
	
	crown={v(10,0,0), v((crown_d_p/2)+b-1,0,0), v((crown_d_p/2)+b-1,h_c,0), v((crown_d_p/2)-1,h_c,0), v((crown_d_p/2),5,0), v(10,5,0), v(10,0,0)} -->Step-1. 
	crown=rotate_extrude(crown, 360) -->Step-1.
	crown=translate(0,0,-cosine_d_p/2-h_c+h_a-1)*crown -->Step-2.
	return crown
end

```
--Function 3: crown generator<br />
--This function executes generation process.<br />
--1.for loop is initiated to iterate for number of teeth of crown gear. shape_precision factor(input parameter) decides the accuracy of gear.<br />
--Following steps are executed in for loop iteratively:<br />
--2.crown workpiece is rotated around Z-axis by small portion'incremental value' of an angle occupied by single tooth.<br />
--3.Cosine pinion is rotated around X-axis by an angle that matches rel. velocity of crown and pinion.<br />
--4.Implement difference of crown workpiece and cosine pinion.<br />
--5.crown workpiece is reallocated with upgraded crown workpiece.<br />
--6.cosine pinion is reallocated with rotated cosine pinion.<br />
--7.Steps "2-6" are repeated iteratively over the range of number of teeth of crown gear.<br />
--8.Smoothness_parameter defines smoothness of gears, High: High precision,  Medium: Medium precision, Low: Low Precision.<br />
```

function crown_generator(z_crown,z_pinion,crown,cosine_pinion)

	smoothness_list={{1, "Medium:0.05"},{2, "High:0.01"},{3, "Low:0.09"},{4, "Default:0.02"}}  -->Step-8.
	smoothness_parameter_input=ui_radio("Shaper precision:", smoothness_list) -->Step-8.
	if smoothness_parameter_input==1 then smoothness_parameter=0.05  -->Step-8.
	elseif smoothness_parameter_input==2 then smoothness_parameter=0.01  -->Step-8.
	elseif smoothness_parameter_input==3 then smoothness_parameter=0.09  -->Step-8.
    elseif smoothness_parameter_input==4 then smoothness_parameter=0.02 end  -->Step-8.
	
	for s=0,z_crown, smoothness_parameter
	do
		generation_angle=((2*math.pi)/z_crown)*s
        generation_angle_1=((2*math.pi)/z_pinion)*s
 --small portion'incremental value' of an angle occupied by single tooth.
		crown_rot_z=rotate(generation_angle,Z)*crown -->step 2.
		shaper_rot_x=rotate(generation_angle_1,X)*cosine_pinion -->step 3-->teeth ratio "z_crown/z_pinion".
		generation=difference(crown_rot_z,shaper_rot_x) -->step 4.
		crown=generation	-->step 5.
		cosine_pinion=shaper_rot_x	-->step 6.	   
	end
	return crown,cosine_pinion
end

```





                                                           -------------END OF FUNCTIONS--------------
```

--INPUT PARAMETERS--	

m_n=ui_text('Enter Module', "4")*1 --Module   Module should be same for meshing gears
b=ui_text('Enter Facewidth', "10")*1  --Face width of crown gear and cosine pinion is same
z_pinion=ui_numberBox("No of Teeth of Pinion Gear", 15.0)*1 --Number of teeth of cosine pinion
z_crown=ui_numberBox("No of Teeth of Crown Gear", 30.0)*1 --Number of teeth of crown
cosine_d_p=m_n*z_pinion --pitch diameter of cosine pinion
crown_d_p=m_n*z_crown  -- pitch diameter of crown

h_a=m_n  --amplitude of cosine profile.

h_c=2*h_a+7 --Height of workpiece of crown gear.Height is maintained below the dedendum of crown gear.
h1=b+5


--COSINE PINION
cosine_pinion=cosine_pinion_maker(m_n,z_pinion,b,crown_d_p)

--CROWN GEAR WORKPIECE
crown=crown_workpiece(b,h_a,crown_d_p,cosine_d_p)

--Note: Meshing position is achieved in such a way that Intersection of perpendicular axes of meshing gears is at origin'world coordinate system'. 
--crown generation process
crown_gear,cosine_pinion=crown_generator(z_crown,z_pinion,crown,cosine_pinion)


---USER CONTROLLED SIMULATION OF MACHINE ELEMENTS:
--Motion is implemented with input parameter from user.
--Input parameter for motion--> motion angle
motion_angle=ui_numberBox("Simulation",0)*(360/z_crown)*0.02 --input parameter.
crown_gear=rotate(motion_angle,Z)*crown_gear		--rotates crown_gear by input parameter.
cosine_pinion=rotate(motion_angle*z_crown/z_pinion,X)*cosine_pinion  --rotates cosine_pinion by rel.velcotiy of crown gear.
emit(cosine_pinion,1)
emit(crown_gear,5)



--screenshot()

```
