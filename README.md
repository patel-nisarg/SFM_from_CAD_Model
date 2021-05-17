# SFM from CAD Model
 3D reconstruction of a 3D printed CAD model.

main.py

- Loads image paths from a text file of images paths. 
**- NOTE: NEW main.py FILE USES 'feature_matches_filtered.npz' FOR DNN FILTERED MATCHES!**
- Initializes camera calibration data
     - For OpenMVG images I posted, calibration data is only K matrix in K.txt
     - No distortion parameters
- Calculate baseline views and triangulate 3D points from baseline
   - return these in a WorldPointsSet object
- Run main SFM loop
- SFM Loop
  - Load image/image features if they have already been generated. If not, generate features using SIFT
  - If view has not had it's 3D points extracted ->update_3D_points(view, ...)
  - Update_3d_points:
    - Compute pose using OpenCV's PnP (code for this in utils.py)
    - If view is in a completed view's tracked points (meaning their 2D point correspondences have been generated), then remove outliers between point correspondences
    - Triangulate points between view and completed views
    - Store 3D points in View object as well as World Points Set object
  - completed_views is a list containing pointers to View objects that have been reconstructed
  - Once 3D points are appended to their respective source views and the world point set, bundle adjustment is applied.

baseline.py

- Feature match the two baseline views 
- Calculate fundamental matrix
- Calculate essential matrix
- Set pose for view1 to be R = I_3 and t = (0, 0, 0)
- Calculate 4 possible poses for view2 using camera_pose_extraction(essential_matrix) found in utils.py
- Triangulate 3D points between view1 and four possible poses of view2
- Disambiguate poses by finding the pose that minimizes reprojection error
- Add 3D points to World Points Set object


view.py

- Contains class for View object
- Features for the image are extracted here (unless features are for baseline views which are extracted in baseline.py)
- Rotation matrix stored in view as (3 x 3). During BA, this is converted to Euler angles vector (3 x 1) using Rodrigues.
- WorldPoints are stored in each View like so: np.array(kp_x, kp_x, point3d_x, point3d_y, point3d_z) -> for an N x 5 array, where N is # of 3D points.
- TrackedPoints are matched points between two images. They are stored for example as view1.tracked_pts = {'view2_id':(view1 kps, view2kps), 'view3_id':(view1kps, view3kps), ..., 'viewn_id':(view1kps, viewnkps)}
   - Each View has an ID associated to it in the form of a hashed string. TrackedPoints takes another View's ID as a key and assigns a tuple containing keypoint matches between the two views.
   - These keypoint matches are used for computing pose via PnP and later filtered using a fundamental matrix between the two views prior to triangulation.

WorldPoints.py

- Stores 3D points, the Views/images that created those 3D points, and the 2D keypoints from the two Views that created it

utils.py


- feature_match
  - Matches features between two views using BF matcher and stores the good results(Lowe's criteria) in view1.tracked_pts and view2.tracked_pts
 
- camera_pose_extraction
  - Takes essential matrix from baseline and returns 4 possible camera poses for second baseline view
  - Checks determinant of rotation matrix to see if it equals 1. If it is -1 then flips sign on rotation matrix and translation vector
  
- pose_disambiguation
  - Checks reprojection error for the 4 poses from second baseline view and triangulated 3D points from each of the four poses
  - returns the 3D points, rotation matrix, and translation vector that minimizes this reprojection error
- triangulate_points
  - Normalizes 2D points in homogeneous coordinates by multiplying them by inverse of intrinsic calibration matrix
  - Triangulates 3D points based on normalized 2D points and projection matrices for two views 
  - returns 3D points as (N x 3)
  
- compute_pose
  - Computes pose for a new view using previously constructed views
  - Feature matches new view with completed views
  - Checks if the matched 2D point from the completed view (view_n) is in the world_points dictionary for that view
  - If the point is in the view's world_points, get the 3D point from that array and store it in local 3D points array. Also store corresponding 2D point in 2D points array. 
  - Calculates pose using cv.solvePnPRansac and the points_3d/points_2d
  
- store_3Dpoints_to_views
  - Stores 3D points to World_points dictonary within the view object.
  - Each 3D point is checked for reprojection error before storing to view's world_points

