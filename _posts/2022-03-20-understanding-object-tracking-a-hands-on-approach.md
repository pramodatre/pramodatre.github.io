---
layout: post
title:  "Understanding Object Tracking: A Hands-on Approach, Part 1"
date:   2022-03-20 06:02:41 -0400
summary: In this post, I will go over object tracking ideas from Forsyth, D., & Ponce, J. (2003). Computer vision A modern approach and implement them in code for deeper understanding.
use_math: true
---
There are multiple state-of-the-art approaches for object detection [^1] [^5]. These approaches addresses the perception problem: How do we perceive various objects in the environment? These objects may move or the perceiver may move around relative to the objects in the real-world. Being able to track these objects reliably over time will allow us to predict their next move--this is crucial for autonomous vehicles among many other applications. Most importantly, tracking objects over time will help us create trajectories of various objects. These trajectories can be analyzes to glean behaviors that will help us move from "perception" to "understanding". Here are some applications that need object tracking: (a) Understanding behaviors of people in enclosed spaces or outdoors for surveillance. (b) Predicting motion of objects for motion planning of an autonomous vehicle (e.g., self-driving cars, drones). (c) Creating object trajectories and provide valuable statistics for different object types (e.g., people and vehicle-type counts, direction of flow of objects).

I was motivated by these applications of object tracking and started reading the chapter on object tracking from Forsyth et. al. [^2]. Here is my attempt to make this chapter more hands-on by applying tracking ideas explained in the chapter to realistic scenes with moving people/objects. I will be presenting some the ideas and corresponding implementation in Python so that you get a good sense of these tracking algorithms. You can apply these ideas to your own projects and start your journey toward "understanding" object behaviors which is quite an exciting topic!

# Dataset Preparation
Detector based tracking approaches assume that the object detector is good enough to detect objects in almost all frames. In this article, we will focus on detector based tracking since we have such good quality detectors at our disposal. Specifically, we will use YOLOv3 [^3] [^4] as our detector and PETS2009 as our dataset for evaluating various approaches to tracking. PETS is an acronym for Performance Evaluation of Tracking and Surveillance. This dataset contains image sequence of people walking around in complex patterns crossing each other and ground-truth bounding boxes around people for all the frames. We will limit our exploration of tracking approaches to the ones that are described in [^2] though there are advances in tracking approaches as described in [^5].

## Clone the repo
First, fetch all the accompanying code from Git using the command:
```shell
git clone https://github.com/pramodatre/cv-algorithms.git
```

## Dataset downloads
PETS2009 dataset is collected with specific tasks such as person count and density estimation, people tracking, and flow analysis and event recognition. We will choose S2L1 dataset which is collected for people tracking containing sparse crowd and difficulty level 1 (L1).
* [PETS2009 image sequence S2L1](http://cs.binghamton.edu/~mrldata/public/PETS2009/S2_L1.tar.bz2)
* [PETS2009 S2L1 ground-truth annotations XML](http://www.milanton.de/files/gt/PETS2009/PETS2009-S2L1.xml)
* [YOLOv3 cfg file](https://github.com/pjreddie/darknet/blob/master/cfg/yolov3.cfg)
* [YOLOv3 weights file](https://pjreddie.com/media/files/yolov3.weights)

You can just run `python prepare_data.py` and the script takes care of downloading and extracting the data for you. If you do not have GPU support on your local machine, it would be too slow to generate detections in real-time using YOLOv3. To circumvent this, we will pre-compute detections and cache them for each image in the ground-truth image-sequence for rapid evaluations and visualization. This will save a lot of time when we wish to visualize results from various tracking approaches.

### Visualizing tracking ground-truth
You can visualize the tracking ground-truth data using the script `play_image_sequence.py`. This script takes image sequence directory location and ground-truth XML file as inputs. The image sequence and corresponding bounding boxes are visualized over time - you will be able to see people moving around with bounding boxes around them. Each person is assigned a unique number as they navigate the scene sometime with tangled paths occluding each other.

{% include image.html img="images/2022-03-20/pets2009S2L1_gt_bboxes.gif" title="PETS ground-truth boxes" caption="Visualizing tracking ground-truth where each person is assigned a unique ID. We will evaluate our tracking algorithms on  this ground-truth data." %}

```shell
python play_image_sequence.py --image_dir './data/Crowd_PETS09/S2/L1/Time_12-34/View_001' --gt './data/PETS2009-S2L1.xml'
```

### Precomputing detections
Running YOLOv3 without GPU support is quite slow. We will pre-compute all the detections per frame and serialize it indexed by the frame number for repeated access. This helps us in rapid testing of our object tracking algorithms and visualize tracking results. All tracking algorithms are implemented in `trackers.py` (runs all trackers on ground-truth when you invoke it). When you invoke `trackers.py` using the following command:

```shell
python trackers.py --image_dir './data/Crowd_PETS09/S2/L1/Time_12-34/View_001'
```

there is this section of code which checks for pre-existing detection file.

```python
if not os.path.exists(saved_detections_file):
    print(
        f"Could not find saved detections file: {saved_detections_file}. Will have to run YOLO on your machine which may be slow the first time. Results will be cached for future runs."
    )
    yolo = YOLOdetector(image_dir)
    yolo.run()
```

If such a file doesn't exist, YOLOv3 model will be used to generate detection per frame and save all the detection and frame index to the file `saved_detections_file`. We will use `trackers.py` as our primary script to write various tracking algorithms.

# Object Tracking Evaluation Metrics
Before we develop object tracking techniques, we need a way to evaluate and quantify the quality of various trackers. This enables us to compare tracking algorithms and also measure the impact of improvements to the tracking algorithm. The computer vision community has realized the complexity of such an evaluation metric and there is a huge body of research dedicated to developing such evaluation metrics for object tracking [^6] [^7]. This is referred to as MOT (Multiple Object Tracking) metrics. There is an effort to create standardized datasets for benchmarking various tracking algorithms. One such popular benchmark is the Multiple Object Tracking Benchmark [^8] [^9]. One such state-of-the-art more recent metric is Higher Order Tracking Accuracy (HOTA) [^11] [^10] which is proposed as an improvement over existing Multiple Object Tracking metrics. HOTA metric measures localization, detection, and association accuracies using an Intersection Over Union (IoU) formulation and combines all these three measures into a single score. A single score for a tracker will help us rank tracking approaches with a unified metric. Further, HOTA metric is interpretable, i.e., when you examine the three scores that are combined to create a single HOTA score, you will be able to narrow down the performance issue of the tracker and take steps to improve the tracking algorithms as described here [^10]. You can read [^10] for an intuitive explanation of HOTA and dig deeper into the metric by reading [^11]. In this post, I will focus on using an open-source implementation of HOTA [^12].

## Computing the HOTA metric
We will leverage an open source implementation of HOTA metrics implemented in Python [^13] for our evaluation. For the sake of completeness, I will go over the steps even though this is just a repetition of details from [^14]. First, let's clone the repo.

```
git clone https://github.com/JonathonLuiten/TrackEval.git
``` 

There is a single script `run_mot_challenge.py` that can run various benchmarks like shown on [^12].

Download the data zip file from [here](https://omnomnom.vision.rwth-aachen.de/data/TrackEval/data.zip). Place the uncompressed file in TrackEval directory (root of the cloned repo). Run the `run_mot_challenge.py` which displays various benchmark evaluation results. This shows that all the files are correctly downloaded and setup.

```shell
python scripts/run_mot_challenge.py --BENCHMARK MOT17 --SPLIT_TO_EVAL train --TRACKERS_TO_EVAL MPNTrack --METRICS HOTA CLEAR Identity VACE --USE_PARALLEL False --NUM_PARALLEL_CORES 1  
```

After you run the above command, you will see various trackers and their evaluation on a benchmark. However, we are interested in running tracking evaluation on our own data, i.e., we would have implemented custom trackers and would like to evaluate their tracking performance on PETS2009 S2L1 ground-truth data. We will have to follow through the instructions outlined here [^14] for computing HOTA metric score. 

At first, it may seem a bit confusing to setup the evaluation directories. However, once you setup the evaluation, you can keep adding your custom trackers to evaluate. I will go through the process so that you don't have to go through the same confusion as I did. Broadly, there are only two steps:
* Setup benchmark, i.e., ground-truth tracking
* Add tracker you want to evaluate

#### Setup benchmark
Ground-truth tracking data is to be placed under `TrackEval/data/gt/mot_challenge/<YourChallenge>`. In our case, let's call the new challenge as `OBJTR22-train`, i.e., create a directory `TrackEval/data/gt/mot_challenge/OBJTR22-train/OBJTR22-01/gt`. You need to place two files in the directory. `gt.txt` (look at ground-truth data preparation section) and `seqinfo.ini` file with the following contents:
```
[Sequence]
name=OBJTR22
seqLength=795
```
PETS2009 S2L1 ground-truth data contains `795` frames hence we have set seqLength to `795`. Next crete three files `OBJTR22-all.txt`, `OBJTR22-train.txt`, and `OBJTR22-test.txt` with the same following content and place them in `TrackEval/data/gt/mot_challenge/seqmaps` directory.
```
name
OBJTR22-01
```
Now you are all set to use this benchmark (ground-truth) to evaluate trackers we will be implementing in this post. 

#### Add tracker you want to evaluate
Create a directory `TrackEval/data/trackers/mot_challenge/OBJTR22-train` where we will place your tracker outputs. To setup our evaluation, we will pretend that we have a perfect tracker, i.e., we will use ground-truth tracking data as our tracker output. Let's call this tracker `PerfectTracker`. To add this tracker for evaluation, first, create `TrackEval/data/trackers/mot_challenge/OBJTR22-train/PerfectTracker/data`. Copy the `gt.txt` to the directory you just created and rename the file to `OBJTR22-01.txt`. 

### Ground-truth data preparation
Ground-truth text file `gt.txt` with ground-truth detections is of the format:
```
<frame>, <id>, <bb_left>, <bb_top>, <bb_width>, <bb_height>, <conf>, <x>, <y>, <z>
```
Here the description from the github page of TrackEval: "The world coordinates x,y,z are ignored for the 2D challenge and can be filled with -1. Similarly, the bounding boxes are ignored for the 3D challenge. However, each line is still required to contain 10 values."

You can export the PETS2009 S2L1 data to the above format by running `python convert_to_mot_challenge_format.py`. The output file `gt.txt` will be written to the current directory. If you would like to confirm if `gt.txt` is correctly exported, you can place the same ground-truth file at `TrackEval/data/gt/mot_challenge/OBJTR22-train/OBJTR22-01/gt` and `TrackEval/data/trackers/mot_challenge/OBJTR22-train/PerfectTracker/data` and run HOTA metric calculation. Rename the file `gt.txt` to `OBJTR22-01.txt` only for the tracker file you place at `TrackEval/data/trackers/mot_challenge/OBJTR22-train/PerfectTracker/data`. So, you will have the same contents of `gt.txt` in the file `TrackEval/data/trackers/mot_challenge/OBJTR22-train/PerfectTracker/data/OBJTR22-01.txt`.

### Run the HOTA evaluation on a perfect tracker
```
python scripts/run_mot_challenge.py --BENCHMARK OBJTR22 --SPLIT_TO_EVAL train --TRACKERS_TO_EVAL PerfectTracker --METRICS HOTA --USE_PARALLEL False --NUM_PARALLEL_CORES 1 

Eval Config:
USE_PARALLEL         : False                         
NUM_PARALLEL_CORES   : 1                             
BREAK_ON_ERROR       : True                          
RETURN_ON_ERROR      : False                         
LOG_ON_ERROR         : /Users/pramodanantharam/dev/git/TrackEval/error_log.txt
PRINT_RESULTS        : True                          
PRINT_ONLY_COMBINED  : False                         
PRINT_CONFIG         : True                          
TIME_PROGRESS        : True                          
DISPLAY_LESS_PROGRESS : False                         
OUTPUT_SUMMARY       : True                          
OUTPUT_EMPTY_CLASSES : True                          
OUTPUT_DETAILED      : True                          
PLOT_CURVES          : True                          

MotChallenge2DBox Config:
PRINT_CONFIG         : True                          
GT_FOLDER            : /Users/pramodanantharam/dev/git/TrackEval/data/gt/mot_challenge/
TRACKERS_FOLDER      : /Users/pramodanantharam/dev/git/TrackEval/data/trackers/mot_challenge/
OUTPUT_FOLDER        : None                          
TRACKERS_TO_EVAL     : ['PerfectTracker']            
CLASSES_TO_EVAL      : ['pedestrian']                
BENCHMARK            : OBJTR22                       
SPLIT_TO_EVAL        : train                         
INPUT_AS_ZIP         : False                         
DO_PREPROC           : True                          
TRACKER_SUB_FOLDER   : data                          
OUTPUT_SUB_FOLDER    :                               
TRACKER_DISPLAY_NAMES : None                          
SEQMAP_FOLDER        : None                          
SEQMAP_FILE          : None                          
SEQ_INFO             : None                          
GT_LOC_FORMAT        : {gt_folder}/{seq}/gt/gt.txt   
SKIP_SPLIT_FOL       : False                         

Evaluating 1 tracker(s) on 1 sequence(s) for 1 class(es) on MotChallenge2DBox dataset using the following metrics: HOTA, Count


Evaluating PerfectTracker

    MotChallenge2DBox.get_raw_seq_data(PerfectTracker, OBJTR22-01)         0.1534 sec
    MotChallenge2DBox.get_preprocessed_seq_data(pedestrian)                0.2807 sec
    HOTA.eval_sequence()                                                   0.3291 sec
    Count.eval_sequence()                                                  0.0000 sec
1 eval_sequence(OBJTR22-01, PerfectTracker)                              0.7656 sec

All sequences for PerfectTracker finished in 0.77 seconds

HOTA: PerfectTracker-pedestrian    HOTA      DetA      AssA      DetRe     DetPr     AssRe     AssPr     LocA      RHOTA     HOTA(0)   LocA(0)   HOTALocA(0)
OBJTR22-01                         100       100       100       100       100       100       100       100       100       100       100       100       
COMBINED                           100       100       100       100       100       100       100       100       100       100       100       100       

Count: PerfectTracker-pedestrian   Dets      GT_Dets   IDs       GT_IDs    
OBJTR22-01                         4650      4650      19        19        
COMBINED                           4650      4650      19        19        

Timing analysis:
MotChallenge2DBox.get_raw_seq_data                                     0.1534 sec
MotChallenge2DBox.get_preprocessed_seq_data                            0.2807 sec
HOTA.eval_sequence                                                     0.3291 sec
Count.eval_sequence                                                    0.0000 sec
eval_sequence                                                          0.7656 sec
Evaluator.evaluate                                                     1.7024 sec
```
Notice that the HOTA score is 100 as we expect as we are comparing ground truth tracking data with ground truth itself.

# Object Tracking Techniques
We will go over some the techniques described for object tracking in Chapter 11 of Forsyth et. al. [^2]. We will implement these strategies in Python to gain deeper understanding of tracking objects in a real-world setting. We will measure the efficacy of each tracking approach using the PETS2009 tracking ground-truth and HOTA metric.

## Baseline tracker
We will use a naive bounding box matching to continue object trajectories as our baseline. In this approach, we will handover bounding boxes from frame $(f-1)$ to frame $f$ using max overlap criteria. That is, we will map a bounding box b1 from frame $(f-1)$ to a bounding box b2 in frame $f$ if b1 intersection b2 is greater than all other bounding box overlaps (assuming there can be multiple bounding boxes between frames $(f-1)$ and $f$). We intentionally use this as our baseline as it is quite straightforward to implement this idea. Here is the implementation of this idea in a method. This is implemented as [BaselineTracker](https://github.com/pramodatre/cv-algorithms/blob/master/object_tracking/trackers.py#L101).

{% highlight python %}
def predict_object_continuation(self, box_t, prev_frame_objects):
    # Select last position for each object and find overlap to box_t
    max_i = 0
    overlap_area = 0
    best_id = -1
    for o_id in prev_frame_objects:
        b = prev_frame_objects[o_id]
        xmin, ymin, xmax, ymax = box_t
        xmin2, ymin2, xmax2, ymax2 = b
        x_overlap = min(xmax, xmax2) - max(xmin, xmin2)
        y_overlap = min(ymax, ymax2) - max(ymin, ymin2)
        # Must check if there is a overlap in x and y direction
        # before computing overlap area. Otherwise, we may end
        # up with +ve area of overlap with both x and y direction
        # overlap is -ve
        if x_overlap > 0 and y_overlap > 0:
            overlap_area = x_overlap * y_overlap
            if overlap_area > max_i:
                max_i = overlap_area
                best_id = o_id
            else:
                continue
    if (max_i > 0):
        return best_id
    return -1
{% endhighlight %}

This method accepts a single bounding box from the current frame and all the bounding boxes from the previous frame as arguments and returns the best object-id in the current frame that is a continuation of the supplied single bounding box. If no such continuation found, this method returns a `-1`.

When you run the baseline tracking algorithm, a file in a suitable format for HOTA evaluation will be written to `tracker_baseline.txt`. Place this file at `TrackEval/data/trackers/mot_challenge/OBJTR22-train/OBJTR22/data`. Rename the file `tracker_baseline.txt` to `OBJTR22-01.txt` and run the  HOTA evaluation script as shown to compute the HOTA metrics.
```
python scripts/run_mot_challenge.py --BENCHMARK OBJTR22 --SPLIT_TO_EVAL train --TRACKERS_TO_EVAL OBJTR22 --METRICS HOTA --USE_PARALLEL False --NUM_PARALLEL_CORES 1 

...

HOTA: OBJTR22-pedestrian           HOTA      DetA      AssA      DetRe     DetPr     AssRe     AssPr     LocA      RHOTA     HOTA(0)   LocA(0)   HOTALocA(0)
OBJTR22-01                         34.46     46.593    25.582    67.789    51.982    28.316    64.04     77.946    41.633    47.954    72.447    34.741    
COMBINED                           34.46     46.593    25.582    67.789    51.982    28.316    64.04     77.946    41.633    47.954    72.447    34.741    

Count: OBJTR22-pedestrian          Dets      GT_Dets   IDs       GT_IDs    
OBJTR22-01                         6064      4650      81        19        
COMBINED                           6064      4650      81        19        
```
The baseline tracker we have implemented has a HOTA score of 34.46 which is computed by combining localization accuracy (LocA), detection accuracy (DetA), and association accuracy (AssA). For deeper intuition of HOTA you can read the paper [^10] and also this well written blog post [^11]. To summarize, our baseline tracker has the following scores.

| Metric| Score|
| **HOTA** | **34.46** |
| LocA | 77.946 |
| DetA | 46.593 |
| AssA | 25.582 |

We will next implement object tracking algorithms mentioned in the book Forsyth, D., & Ponce, J. (2003) [^3] and evaluate their performance using the HOTA metrics.

## Detection based tracking
If we have a good detector, we can use detection based tracking. "Good" detector seems subjective but this approach assumes that each object is detected reliably across frames. Detection based tracking then bridges object movements across frames by matching detections from one frame to the next frame. The matching is done using overlap score and using a bipartite matching to ensure we assign one detection in a frame to a single detection in the next frame.

Here is the tracking by detection Algorithm 11.1 from the book Forsyth, D., & Ponce, J. (2003) [^3]:

---
Notation:
* $x_{k}$(t) is the $k^{th}$ response of the detector in the $i^{th}$ frame
* t(k, i) is the $k^{th}$ track in the $i^{th}$ frame
* *t(k, i) is the detector response attached to the $k^{th}$ track in the $i^{th}$ frame

Assumption: 
* We have a reasonably reliable detector with distance d such that d(*t(k, i), *t(k, i-1)) is small. In words, the detector response in $i^{th}$ frame and the previous frame $(i-1)^{th}$ frame are quite close.

```
First frame: 
> Create a track for each detector response

All other frame:
> Link tracks and detector responses by bipartite matching
> Spawn a new track for each detector response not assigned to a track
> Prune any track that has not received a detector response for some frames

Cleanup: 
> We now have trajectories in space time. Link trajectories when 
justified (perhaps using a dynamical or appearance model)   
```
---
The above algorithm is implemented as [DetectionBasedTracker](https://github.com/pramodatre/cv-algorithms/blob/master/object_tracking/trackers.py#L248).

If the notation above is confusing, no worries! I would just think in terms of object detections in each frame. We need to assign a detection in a frame to a detection in the next frame -- this enables us to track object detection across frames. Here is a method that does exactly this.

```python
def predict_object_continuation_using_bipartite_matching(self, cur_det,obj_map):
    """Connect objects in previous frame (obj_map) to objects in
    the current frame (cur_det) using an optimization technique.
    Args:
        cur_det (list): Containing Detection objects; one
            detection object per detection
        obj_map (dict): Containing object_id as key and
            corresponding bounding box as value
    Returns:
        dict: Updated obj_map
    """
    if not obj_map:
        # First frame
        for det in cur_det:
            obj_map[self.o_id_count] = det.get_xmin_ymin_xmax_ymax()
            self.o_id_count += 1
    else:
        # Rest of the frames
        cur_det_dict = {}
        rows, cols = len(list(obj_map.keys())), len(cur_det)
        cost_matrix = np.zeros((rows, cols))
        cost_martix_df = pd.DataFrame(
            data=cost_matrix, index=list(obj_map.keys()), columns=list(range(cols))
        )
        # read all detections to a dictionary
        det_count = 0
        for c_det in cur_det:
            det_bbox = c_det.get_xmin_ymin_xmax_ymax()
            print(det_bbox)
            cur_det_dict[det_count] = det_bbox
            det_count += 1
        for o_id in obj_map:
            for det_id in cur_det_dict:
                det_bbox = cur_det_dict[det_id]
                iou_score = self.compute_iou_score(obj_map[o_id], det_bbox)
                if iou_score > 0:
                    cost_martix_df.loc[o_id, det_id] = iou_score
        cost_martix_array = cost_martix_df.values * -1
        row_ind, col_ind = linear_sum_assignment(cost_martix_array)
        # Update object map with best assignments
        for i, j in zip(row_ind, col_ind):
            obj_id = list(cost_martix_df.index)[i]
            det_id = cost_martix_df.columns[j]
            if cost_martix_array[i, j] == 0:
                obj_map[self.o_id_count] = cur_det_dict[det_id]
                self.o_id_count += 1
            else:
                obj_map[obj_id] = cur_det_dict[det_id]
    return obj_map
```
This method predicts object continuations from one frame to the next using an optimization approach called Hungarian Optimization a.k.a. linear sum assignment or bipartite matching. I wanted to point out some important implementation details here. For complete code, please refer to [github repo](https://github.com/pramodatre/cv-algorithms/blob/master/object_tracking/trackers.py#L248). We maintain a dictionary for any object that is currently in the frame. The dictionary has object ID as key and its bounding box as value. This dictionary is updated within this method and the updated dictionary is returned.

We are using a global counter (self.o_id_count) which is a class attribute to track object ID and increment it for new object ID assignments. A new object ID is assigned to a detection in the current frame when the cost matrix has zero entry for all object bounding boxes in the previous frame. This may need a bit of explanation! The cost matrix is updated using the Intersection over Union (IoU) scores between an object detection bounding box in the previous frame and all object detection bounding boxes in the current frame. Since the optimization minimizes overall cost of assigning objects from the previous frame to the objects in current frame, we will have to use a negative of the IoU scores. That is, we prefer object assignments with highest IoU scores and this translates to the least cost when we take negative of IoU scores.

The cleanup step in the above algorithm description is implemented in `prune_tracks` method as shown below. Notice that we had to pick number of frames for which an object position was not updated to prune them -- set as a global variable `STALE_DET_THRESHOLD_FRAMES`.

```python
def prune_tracks(self, cur_obj_map, prev_obj_map):
    """Remove objects whose positions are not updated for certain frames.

    Args:
        cur_obj_map (dict): Object id and corresponding
                    detection bounding box
        prev_obj_map (dict): Previous frame object id and
                     corresponding bounding box

    Returns:
        dict: Pruned cur_obj_map
    """
    print(f"comparing {cur_obj_map} and {prev_obj_map}")
    for cur_id in cur_obj_map:
        if cur_id in prev_obj_map:
            if cur_obj_map[cur_id] == prev_obj_map[cur_id]:
                self.o_ids_without_updates_counts[cur_id] += 1

    keys_to_remove = []
    for cur_id in self.o_ids_without_updates_counts:
        if (
            self.o_ids_without_updates_counts[cur_id]
            > self.STALE_DET_THRESHOLD_FRAMES
        ):
            keys_to_remove.append(cur_id)

    for cur_id in keys_to_remove:
        del cur_obj_map[cur_id]
        del self.o_ids_without_updates_counts[cur_id]

    return cur_obj_map
```

Here is the HOTA evaluation for the implementation of detection based tracking approach.

```
python scripts/run_mot_challenge.py --BENCHMARK OBJTR22 --SPLIT_TO_EVAL train --TRACKERS_TO_EVAL TrackByDetection --METRICS HOTA --USE_PARALLEL False --NUM_PARALLEL_CORES 1 

Eval Config:
USE_PARALLEL         : False                         
NUM_PARALLEL_CORES   : 1                             
BREAK_ON_ERROR       : True                          
RETURN_ON_ERROR      : False                         
LOG_ON_ERROR         : /Users/pramodanantharam/dev/git/TrackEval/error_log.txt
PRINT_RESULTS        : True                          
PRINT_ONLY_COMBINED  : False                         
PRINT_CONFIG         : True                          
TIME_PROGRESS        : True                          
DISPLAY_LESS_PROGRESS : False                         
OUTPUT_SUMMARY       : True                          
OUTPUT_EMPTY_CLASSES : True                          
OUTPUT_DETAILED      : True                          
PLOT_CURVES          : True                          

MotChallenge2DBox Config:
PRINT_CONFIG         : True                          
GT_FOLDER            : /Users/pramodanantharam/dev/git/TrackEval/data/gt/mot_challenge/
TRACKERS_FOLDER      : /Users/pramodanantharam/dev/git/TrackEval/data/trackers/mot_challenge/
OUTPUT_FOLDER        : None                          
TRACKERS_TO_EVAL     : ['TrackByDetection']          
CLASSES_TO_EVAL      : ['pedestrian']                
BENCHMARK            : OBJTR22                       
SPLIT_TO_EVAL        : train                         
INPUT_AS_ZIP         : False                         
DO_PREPROC           : True                          
TRACKER_SUB_FOLDER   : data                          
OUTPUT_SUB_FOLDER    :                               
TRACKER_DISPLAY_NAMES : None                          
SEQMAP_FOLDER        : None                          
SEQMAP_FILE          : None                          
SEQ_INFO             : None                          
GT_LOC_FORMAT        : {gt_folder}/{seq}/gt/gt.txt   
SKIP_SPLIT_FOL       : False                         

Evaluating 1 tracker(s) on 1 sequence(s) for 1 class(es) on MotChallenge2DBox dataset using the following metrics: HOTA, Count


Evaluating TrackByDetection

    MotChallenge2DBox.get_raw_seq_data(TrackByDetection, OBJTR22-01)       0.1711 sec
    MotChallenge2DBox.get_preprocessed_seq_data(pedestrian)                0.2702 sec
    HOTA.eval_sequence()                                                   0.3073 sec
    Count.eval_sequence()                                                  0.0000 sec
1 eval_sequence(OBJTR22-01, TrackByDetection)                            0.7511 sec

All sequences for TrackByDetection finished in 0.75 seconds

HOTA: TrackByDetection-pedestrian  HOTA      DetA      AssA      DetRe     DetPr     AssRe     AssPr     LocA      RHOTA     HOTA(0)   LocA(0)   HOTALocA(0)
OBJTR22-01                         39.458    48.459    32.238    67.952    53.977    39.871    49.078    77.514    46.798    55.504    71.343    39.598    
COMBINED                           39.458    48.459    32.238    67.952    53.977    39.871    49.078    77.514    46.798    55.504    71.343    39.598    

Count: TrackByDetection-pedestrian Dets      GT_Dets   IDs       GT_IDs    
OBJTR22-01                         5854      4650      83        19        
COMBINED                           5854      4650      83        19        

Timing analysis:
MotChallenge2DBox.get_raw_seq_data                                     0.1711 sec
MotChallenge2DBox.get_preprocessed_seq_data                            0.2702 sec
HOTA.eval_sequence                                                     0.3073 sec
Count.eval_sequence                                                    0.0000 sec
eval_sequence                                                          0.7511 sec
Evaluator.evaluate                                                     1.3774 sec
```
Observe that there are improvements to tracking throughout! Here is the evaluation summary with percentage improvements shown from the baseline tracker. Biggest gain is in association accuracy which makes sense since the detector based approach is using an optimization approach to assign object continuations between frames minimizing the overall cost of assignments. The baseline approach is doing this assignment without using a greedy approach by selecting the highest overlapping bounding box across frames.

| Metric| Score|
| **HOTA** | **39.45** ($$ \uparrow $$ 14%)|
| LocA | 77.514 ($$ \downarrow $$ 0.5%)|
| DetA | 48.459 ($$ \uparrow $$ 4%)|
| AssA | 32.238 ($$ \uparrow $$ 26%)|

# Conclusion
In this post, you were able to understanding basic tracking strategies, setup ground-truth data, select evaluation metrics, and implement BaselineTracker and DetectionBasedTracker in Python. You are able to quantify the improvements you made from baseline tracker to detection based tracker using HOTA metrics. Having such a metric that measure our progress as we refine our approach is crucial for staying focused in our attempt to improve our approach to object tracking. This metric provide direct feedback to us allowing us to measure success of new ideas and thereby iterate quickly to improve our approach. In future posts, I will go though the implementation of more tracking approaches such as matching by pixel similarity and motion models presented in Forsyth et. al. [^2].

**Code used in this blog post is [here](https://github.com/pramodatre/cv-algorithms/tree/master/object_tracking)**

# References
---
[^1]: [Object Detection Papers with Code](https://paperswithcode.com/task/object-detection)
[^2]: Forsyth, D., & Ponce, J. (2003). Computer vision: A modern approach. Upper Saddle River, N.J: Prentice Hall.
[^3]: Redmon, J., & Farhadi, A. (2018). YOLOv3: An Incremental Improvement. ArXiv, abs/1804.02767.
[^4]: [YOLO: Real-Time Object Detection](https://pjreddie.com/darknet/yolo/)
[^5]: [Object Detection and Tracking in 2020](https://blog.netcetera.com/object-detection-and-tracking-in-2020-f10fb6ff9af3)
[^6]: Yin, Fei & Makris, Dimitrios & Velastin, Sergio. (2007). Performance evaluation of object tracking algorithms. 10th IEEE International Workshop on Performance Evaluation of Tracking and Surveillance (PETS2007). 
[^7]: Luiten, J., Os̆ep, A., Dendorfer, P. et al. HOTA: A Higher Order Metric for Evaluating Multi-object Tracking. Int J Comput Vis 129, 548–578 (2021). https://doi.org/10.1007/s11263-020-01375-2
[^8]: [Multiple object Tracking Benchmark](https://motchallenge.net/)
[^9]: [Jonathon Luiten, Arne Hoffhues, TrackEval](https://github.com/JonathonLuiten/TrackEval)
[^10]: [How to evaluate tracking with the HOTA metrics](https://jonathonluiten.medium.com/how-to-evaluate-tracking-with-the-hota-metrics-754036d183e1)
[^11]: Jonathon Luiten, A.O. & Leibe, B. HOTA: A Higher Order Metric for Evaluating Multi-Object Tracking. International Journal of Computer Vision, 2020.
[^12]: [MOT Challenge Metrics Implementation](https://github.com/JonathonLuiten/TrackEval/tree/master/docs/MOTChallenge-Official)
[^13]: [HOTA Metric Implementation in Python](https://github.com/JonathonLuiten/TrackEval/blob/master/trackeval/metrics/hota.py)
[^14]: [Tracking Evaluation on Your Own Data](https://github.com/JonathonLuiten/TrackEval/tree/master/docs/MOTChallenge-Official#evaluating-on-your-own-data)

