from imaging_interview import preprocess_image_change_detection as picd
from imaging_interview import compare_frames_change_detection as cfcd
import os
import cv2

input_F = r"C:\Users\dhadu\OneDrive\Desktop\task\dataset-candidates\dataset"

# list only the files in this folder with png extension
files = [x for x in os.listdir(input_F) if x.endswith('png')]

# create the list of camera numbers
# to later organise the files acc. to camera numbers
camera_num = list()
for cf in range(len(files)):

    if '-' in files[cf]:
        cam = files[cf].split('-')[0]
    else:
        cam = files[cf].split('_')[0]

    if cam not in camera_num:
        camera_num.append(cam)

# organise files according to camera number
# as to easy the process of comparing the images
# presumably images from one camera will not be similar to images from other camera
org_files = list()
for cn in camera_num:
    temp_list = list()

    for cf in files:
        if cf.startswith(cn):
            temp_list.append(cf)

    org_files.append(temp_list)

# loop over sublists of org_files list
org_delete_list = list()

for ln in range(len(org_files)):
    deleted_list = list()

    # loop over images of camera 1, camera 2, camera 3 and so on...
    for img_num in range(len(org_files[ln])):

        # fetch name of image 1
        img_1 = org_files[ln][img_num]

        # continue to next image if this image name is in deleted_list
        # this image is already deleted from directory due to similarity with another file
        # so no need to process further
        if img_1 in deleted_list:
            continue

        # define the path to image 1
        img_1_path = os.path.join(input_F, img_1)
        if not os.path.isfile(img_1_path):
            continue

        # open read image into array form with cv2.imread
        img_1_cv = cv2.imread(img_1_path)
        if img_1_cv is None:
            continue

        # preprocess i.e. first image with given definition "preprocess_image_change_detection"
        # for comparison with second image
        gray_1 = picd(img_1_cv)

        # loop over rest of images of camera 1, camera 2, camera 3 and so on...
        for nimg_num in range(len(org_files[ln][img_num + 1:])):

            # fetch name of image 2
            img_2 = org_files[ln][img_num + 1:][nimg_num]

            # continue to next image if this image name is in deleted_list
            # this image is already deleted from directory due to similarity with another file
            # so no need to process further
            if img_2 in deleted_list:
                continue

            # define the path to image 2
            img_2_path = os.path.join(input_F, img_2)
            if not os.path.isfile(img_2_path):
                continue

            # open read image into array form with cv2.imread
            img_2_cv = cv2.imread(img_2_path)
            if img_2_cv is None:
                continue
            # preprocess i.e. second image with given definition "preprocess_image_change_detection"
            # for comparison with first image
            gray_2 = picd(img_2_cv)

            # this condition is important because
            # when the images are read into array their array sizes are different according to their image size
            # so if this condition is not used then cv2.abs diff will return error
            # because cv2.abs diff differentiate between array of same type and same size
            if gray_1.size == gray_2.size:

                # compare the preprocessed image 1 and image 2 with minimum contour area 0
                # min contour area is taken as 0 because of desire to not neglect even smallest possible area of an image
                compare_result_tuple = cfcd(gray_1, gray_2, 0)

                # if the score of image comparison is some integer or below
                # then delete image 2 as it is similar to image 1
                # but it is hard to decide upon this integer number
                # the reason is different light exposure
                # in definition compare_frames_change_detection only basic THRESH_BINARY is used
                # which converts light exposed area into a big contour
                # even if the images are similar in terms of objects but the light exposure is different than
                # score of these compared images will be much higher, so it is difficult to decide upon one integer
                # on the other hand if adaptive thresholding is used when there is a different light exposures in different areas
                # score of compared images can probably be reduced and this score of difference will be
                # in terms of difference between objects not the light exposure

                # compare the score of image difference with the 15% of preprocessed image size
                # the reason for that is if score is compared to one constant smaller integer
                # then for bigger sized images even the minor difference in lightning will give very big score number compared
                # to smaller images with the same lightning difference
                # thus this score should be compared to constantly changing integer according to image size
                if compare_result_tuple[0] <= gray_1.size*0.05:
                    deleted_list.append(img_2_path)
                    os.remove(img_2_path)

                # if calculated score is not decided integer or below then continue to next image
                else:
                    continue

            # if image 1 and image 2 array sizes are not same then continue
            else:
                continue

    # append deleted list to org_delete_list
    # as to confirm after wards which images from which camera number is deleted
    org_delete_list.append(deleted_list)

del_img_file = os.path.join(input_F, "delete_images_list.txt")
with open(del_img_file, 'w') as diFile:
    for nm in range(len(org_delete_list)):
        diFile.write(camera_num[nm]+'\n')
        for di in org_delete_list[nm]:
            diFile.write(di+'\n')

