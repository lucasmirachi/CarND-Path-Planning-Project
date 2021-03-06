# Writeup - CARN Path Planning Project
---

The goal of this project is to safely navigate around a virtual highway with other traffic that is driving +-10 MPH of the 50 MPH speed limit. The simulator environment provide the car's localization and sensor fusion data, and there is also a sparse map list of waypoints around the highway. So, in order to achieve this goal, it was asked to build a path planner that creates smooth, safe trajectories for the car to follow, while respecting the following limits:

* The **speed goal** is to have the car traveling at (but not above) the 50 MPH speed limit as often as possible (but there will be times when traffic gets in the way).
* The car should be able to make one complete loop around the 6946m highway.
* In order to provide an enjoyable ride for the vehicle's passenger, both the **jerk** and the **total acceleration** should not exceed 10 m/s^2. 
* The car should avoid hitting other cars at all cost as well as driving inside of the marked road lanes at all times, unless going from one lane to another. 
* The car should try to go as close as possible to the 50 MPH speed limit, which means passing slower traffic when possible, note that other cars will try to change lanes too. 
* The car should only change lanes if such a change would be safe, and also if the lane change would help it move through the flow of traffic better. *

In order to control the car movement, the simulator adopts the **Point Paths** system. Basically, the path planner outputs a list of x and y global map coordinates where each pair of x and y coordinates is a point, and all of the points together form a trajectory. Every 20 ms the car moves to the next point on the list. The car's new rotation becomes the line between the previous waypoint and the car's new location, just like the animation bellow shows.

<p align="center">
<img src="./writeup_imgs/car-ppc-gif.gif" alt="car-ppc" width="350"/>
</p>

The simulator is telling us exactly where the "x, y" and "s, d" coordinates of the car are at, along with the car's angle and the car's speed.

### Moving the ego-vehicle straight

When first opening the simulator, I faced a immobile vehicle in the center of a road since it has an empty list of x and y points and the simulator couldn't define where in the cartesian plan the car is supposed to go next.

```
msgJson["next_x"] = next_x_vals;
msgJson["next_y"] = next_y_vals;
```

So, as a first step, I'll start moving the vehicle forward by setting the distance increment (dist_inc) to 0.5, which is how much the points are going to be spaced apart. Since the car moves 50 times a second, a distance of 0.5m per move will create a velocity of 25 m/s. 25 m/s is close to 50 MPH.

To do so, I passed the **next_x_vals** and **next_y_vals** to the simulator by using the following code:

```
double dist_inc = 0.5;
for (int i = 0; i < 50; ++i) {
  next_x_vals.push_back(car_x+(dist_inc*i)*cos(deg2rad(car_yaw)));
  next_y_vals.push_back(car_y+(dist_inc*i)*sin(deg2rad(car_yaw)));
}
```

<p align="center">
<img src="./writeup_imgs/path-planning-1.gif" alt="gif1" width="450"/>
</p>

Despite being able to make the car drive forward at constant velocity, I made the car go from 0 MPH to 56 MPH in a single 20ms frame, causing a spike in acceleration!  

According to Udacity's class notes, *"acceleration is calculated by comparing the rate of change of average speed over .2 second intervals. In this case total acceleration at one point was as high as 75 m/s^2. Jerk was also very high. The jerk is calculated as the average acceleration over 1 second intervals. Part of the total acceleration is the normal component, AccN which measures the centripetal acceleration from turning. The tighter and faster a turn is made, the higher the AccN value will be".*

So now that I made a car that is able to drive in a straight line, now I should try to develop something to better understand what can be done so the car will keep in lanes.

To do so, I'm going to make use of <mark>Frenet coordinates</mark>, which are a way of representing position on a road in a more intuitive way than traditional (x,y)(x,y)(x,y) Cartesian Coordinates. 

With Frenet coordinates, we use the variables **s** and **d** to describe a vehicle's position on the road. The s coordinate represents distance along the road (also known as **longitudinal displacement**) and the d coordinate represents side-to-side position on the road (also known as **lateral displacement**).

<p align="center">
<img src="./writeup_imgs/frenet.png" alt="frenet" width="350"/>
</p>

To convert the cartesian coordinates to Frenet coordinates, I used the helper function **getFrenet**, which takes in cartesian (x,y) coordinates and transforms them to Frenet(s,d) coordinates. The opposite conversion may be done by using the **getXY** helper function.

With the following code line, for each increment in time, the car will be following a **s** coordinate waypoint.  For next_d, because we're initially in middle lane (and because the waypoints are measured from the double yellow line in the middle of the road), we're technically one and a half lanes from where the waypoints are. So, once the lanes are 4 meters wide, our next_d will be 4+ 2 = 6!


```
double next_s = car_s + (i + 1) * dist_inc;

double next_d = 6;
```

Now we'll be ready to create a vector containing X and Y, and to do so, I'll make use of getXY function:

```
vector<double> xy = getXY(next_s, next_d, map_waypoints_s, map_waypoints_x, map_waypoints_y);

next_x_vals.push_back(xy[0]); //s
next_y_vals.push_back(xy[1]); //d
```

<p align="center">
<img src="./writeup_imgs/path-planning-2.gif" alt="gif2" width="450"/>
</p>

Despite being able to make the car drive inside the middle lane, we can notice that it has a kind of robotic movement while making curves, which violates our jerk limit! 

In order to improve the car movement and smooth out this kind of disjointed path that the car is following in the simulator (so that it might not violate the acceleration and jerk limits), I'll be using the **spline** library [lines 227 to 265] which, by adding previous path points and calculating the distance "y position" 30 meters ahead, it was possible to achieve a smoother driving!  

<p align="center">
<img src="./writeup_imgs/spline.gif" alt="gif3" width="450"/>
</p>

### Dealing with other vahicles on the road - Prediction & Behavior Planning

In order to start working with sensor fusion and start taking in account the other cars on the road. Since the simulator is reporting in a list containing the position and movement information of all the other cars on the road (s, d, x, y, vx and vy values) and we want to use that to try to figure out where the other cars actually are, how fast they are going, and how we should behave as result.

To do so, first I need to check if there is a car ahead in my lane and then, if it is in my lane, we gotta check the speed of the car (pulling that from vx and vy, which is the third and fourth elements of sensor_fusion vector) and check also the car's s value (which is the fifth element of sensor_fusion vector). In this way, we can forecast that, in the future, if the other car's s is less than 30 meters apart of our car's s, we're gonna take some action. 

```
//find ref_v to use
          for (int i = 0; i < sensor_fusion.size(); i++){
            //car is in my lane
            float d = sensor_fusion[i][6];
            if (d < (2 + 4 * lane + 2) && d > (2 + 4 * lane - 2)){
                double vx = sensor_fusion[i][3];
                double vy = sensor_fusion[i][4];
                double check_speed = sqrt(vx * vx + vy * vy);
                double check_car_s = sensor_fusion[i][5];

                check_car_s += ((double)prev_size * 0.2 * check_speed); //if using previous points can project s value out
                //check s values greater than mine and s gap
                if ((check_car_s > car_s) && ((check_car_s - car_s) < 30)){
                    
                }
            }
          }
```

Now that we have a very smooth car movement, it is time to solve the "cold start" issue (where in the start of the simulation we basically go from 0 to 50MPH almost instantaneously).

To do that, I changed the start velocity to 0MPH (previously 50PMH) and then, by using the same *"too_close"* code from the sensor fusion part, the car will automatically notice that, in the beginning, there will be no other car in front of our ego-vehicle, so it will start incrementing the velocity (using increments of .224) until it achieves a max_velocity of around 49.5MPH!

```
if(too_close)
{
  ref_vel -= .224;
}
else if(ref_vel < 49.5){
  ref_vel += .224;
}
```

In short, the **behavioral planning** component  determines what behavior the vehicle should exhibit at any point in time (for example: stopping at a traffic light or intersection, changing lanes, accelerating, or making a left turn onto a new street are all maneuvers that may be issued by this component). Looking at the overall flow of data in a self-driving car in the picture below,  we can see that behavior planning receives inputs from the prediction module and localization module (both of which get their inputs from sensor fusion). Meanwhile, the output from the behavior module goes directly to the trajectory planner (which also takes input from prediction and localization so that it can send trajectories to the motion controller). So, everything inside this <mark>highlighted</mark> box is commonly referred to as Path Planning.


<p align="center">
<img src="./writeup_imgs/behavior.png" alt="behavior_planning" width="350"/>
</p>

Meanwhile, the **prediction** component estimates what actions other objects might take in the future. For example, if another vehicle were identified, the prediction component would estimate its future trajectory. In this example prediction module we use to find out following car ahead is too close, car on the left is too close, and car on the right is too close.
 
 <p align="center">
<img src="./writeup_imgs/prediction.png" alt="prediction" width="200"/>
</p>
 
As explained, actual prediction module will be implemented using the approach mentioned above, but this highway project doesn't need to predict the trajectory of each vehicle as those vehicles trajectory will be on the straight lane most of the time.

The final prediction code then became the following:

```
 if(prev_size > 0) {
	car_s = end_path_s;
}

bool car_left= false;
bool car_right = false;
bool car_ahead = false;
for(int i=0; i < sensor_fusion.size(); i++) {
	float d = sensor_fusion[i][6];
	int check_car_lane;
	/*Currently, we assume that we only have three lanes and each lane has 4 meter width. In actual scenario, number of lanes and distance between the lanes and total lanes distance can be detected using computer vision technologies. 
*/

	if(d > 0 && d < 4) {
		check_car_lane = 0;
	} else if(d > 4 && d < 8) {
		check_car_lane = 1;
	} else if(d > 8 and d < 12) {
		check_car_lane = 2;
	}   
                            
	double vx = sensor_fusion[i][3];
	double vy = sensor_fusion[i][4];
	double check_speed = sqrt(vx*vx+vy*vy);
	double check_car_s = sensor_fusion[i][5];   
	//This will help to predict where the vehicle will be in future
	check_car_s += ((double)prev_size*0.02*check_speed);
	if(check_car_lane == lane) {
		//A vehicle is on the same lane and check the car is in front of the ego car
		car_ahead |= check_car_s > car_s && (check_car_s - car_s) < 30;                                     
		} else if((check_car_lane - lane) == -1) {
			//A vehicle is on the left lane and check that is in 30 meter range
			car_left |= (car_s+30) > check_car_s  && (car_s-30) < check_car_s;
		} else if((check_car_lane - lane) == 1) {
			//A vehicle is on the right lane and check that is in 30 meter range
			car_right |= (car_s+30) > check_car_s  && (car_s-30) < check_car_s;
                            
		}
	}

```

And the final output inside the simulator:

<p align="center">
<img src="./writeup_imgs/path-planning-final.gif" alt="gif4" width="550"/>
</p>

Finally, the car is now able to drive for more than 10 minutes without having any incident (being able to even drive all around the highway track while controling the velocity and changing lanes only when it is safe enough)!