# REU 2018 - Evaluating the Role of Audio in Comprehensive Video Understanding
[Justin Goodman](https://terpconnect.umd.edu/~jugoodma) & Marc Moore

## Note before we begin
The processes for creating mturk HITs are mechanical and can absolutely be automated, but we like using mturk's front-end interface to create HIT batches. Mturk has an [API](https://docs.aws.amazon.com/AWSMechTurk/latest/AWSMturkAPI/Welcome.html) which we tried using. The issue with it is that we have to keep track, on our own, of active HITs. It is easy to create HITs but difficult to manage them because we would have to build a whole front-end interface. There's no point in re-inventing the wheel -- Amazon has a front-end already built.

Okay, on to the fun stuff!

## Temporal Annotations
Part 1 of this project is expanding the [AVE dataset](https://sites.google.com/view/audiovisualresearch) led by [Yapeng Tian](http://yapengtian.org/). Given a label for a 10-second video, we need to annotate _when_ the object described by that label makes sound. The dataset is a subset of [Google's AudioSet](https://research.google.com/audioset/download.html). AVE started out with 4143 videos spanning 28 categories. We are expanding the set to contain 10000 videos spanning over 50 categories.

AVE was created with in-house annotators (paid undergraduate students). To gather the remaining 6000 videos we decided to use [Amazon Mechanical Turk](https://www.mturk.com). We built a temporal annotation interface. The following steps explains how to use our code with mturk.

1. Download Google's AudioSet (links at the bottom) unbalanced train segments and place `.csv` file in `/data`
1. Download Google's Ontology `.json` file describing each label and place said file in `/data`
1. Figure out what labels you want to run. Write each label in a file titled `temporal-input-labels.txt` in `/data`. Each label should be separated by a newline. An example file is provided
1. Figure out how many videos per category you want. The default is 200, but you can specify a different amount in the first command line argument you provide to the script
1. Run `temporal-input.py` in the `/scripts` directory. This will output all the videos into `/input/temporal-input.csv`. If this fails, then files are not placed correctly, or one of your chosen labels does not have enough videos
1. Find a server to host your videos. Create a directory with all the videos. Make sure each video has the format `ytid.mp4`. Alternatively, you can use our current template which utilizes the YouTube player
1. Create a [mturk project](https://www.requester.mturk.com/create). Choose 'other'. Specify your settings on the first tab. Paste in our template `/templates/temporal-ui.html` on the second tab. If you are hosting the videos yourself, put your server in a `<source>` tag within a `<video></video>` tag instead of the YouTube `<iframe></iframe>`. Save project
1. Create a new batch for this project and upload the `temporal-input.csv` generated earlier
1. Run batch and wait for results. Because of the price and nature these HITs get completed super fast. We have a few workers who do an excellent job with these
1. We do not have a visualizer for these data. See the next paragraph on how to Approve and Reject HITs
1. Download batch results as `.csv` and place file in `/results`
1. We are still working on a script that creates a `.csv` from the temporal data -- stay tuned

To start, let's introduce what is included in a response. Temporal annotations include a link to the HIT, everything that was inputted to create the HIT, whether the worker clicked the instructions, whether the worker clicked play, if the input video contains an AVE, and the actual annotation (0.25-second interval answer boxes for the duration of the max 10-second video). Non-text responses are generally a 1 or a 0 representing yes or no respectively. For the 'contains' response, 1 represents 'contains AVE', 0 represents 'does not contain AVE', -1 represents 'only see the label', and -2 represents 'only hear the label'. On to approving and rejecting. There's a lot of discretion involved with checking temporal annotations. First thing to check is the approval rate. If a turker has never done a hit, we check all their work or enough to be satisfied that they can/can't do these tasks. If some responses are good and some are bad you'll have to make the call if it's worth your time to check them all or block the worker. Second thing to check is response type. If every response for the type (whether it contains an AVE) has all 1's then it's unlikely to be accurate. From a random sample we should see SOME distribution of 1s, 0s, -1s, and -2s. Third, check for annotation patterns. If a worker just clicks one temporal location, or if a worker selects all temporal locations, or if you see all 0s with a few 1s sprinkled in, then it's unlikely to be accurate and you should check these. Fourth, check HITs where the video wasn't watched. Sometimes the javascript code just runs slow and doesn't catch when the user clicks play, but that happens less than usual. Usually good workers with high approval rates do accurate work, but sometimes they just mess up certain things. Maybe someone doesn't know what a banjo is so they mark 0 for everything with the label 'banjo' -- who knows. This is a time-consuming process but the accuracy is `>=` the accuracy of the method used to gather the original AVE dataset. Lastly, if the worker's approval rate is 0 and they did not click on the instructions then their work is fishy and unlikely to be correct (but again, the javascript could have just been slow to catch the instructions).

## Spatial Annotations
Part 2 of this project is creating a validation dataset for the attention maps generated by Yapeng Tian's attention model (described in his AVE paper). To the best of our knowledge, this is a unique dataset. Given the validation set for the AVE dataset (containing a video with a label and AVE temporal boundary), we need to annotate _where_ the object described by the label makes the sound. We do this by showing mturk workers a 1-second clip, played on repeat, asking them to color in a grid overlaid on the clip. Each AVE is max 10-seconds, which means the worst-case number of HITs we create is `(num vids) * 10`. When AVE gets to 10000 videos, the validation set will be roughly 1000 videos, which equates to worst-case 10000 annotations. We have collected annotations for all 402 validation videos in the original AVE dataset. As we run more temporal annotation HITs we will need to run more spatial annotation HITs.

To gather these spatial annotations we, of course, use Amazon Mechanical Turk. We built a spatial annotation interface. The video is displayed at resolution `640x360 pixels`, and the grid overlay is `32x18 boxes`. Each box is a `20x20 pixel` square. We have the annotations stored in both `.json` (described below) and `.csv`. The `.json` has the actual grid responses, whereas the `.csv` translates the grid response to bounding boxes. Our spatial visualizer shows the bounding boxes, and our approve/reject visualizer tool shows the actual response.

1. Download AudioSet and the Ontology
1. Download the AVE validation set
1. ...
1. ...

The data is an array of objects. Each object has a YouTube ID, a start time and end time described by Google's AudioSet, a start and end time of the audio-visual event, and a list of the vectorized grid annotations. There are `32x18 = 576` grid boxes, so each vector contains 576 integers. 0 means the box was NOT clicked, and 1 means the box WAS clicked. The length of the annotations vector is equivalent to `ave end - ave start` (and each annotation is in order temporally). Here's the file structure:
```json
[
	{
		"ytid": str,
		"start time": float,
		"end time": float,
		"ave start": int,
		"ave end": int,
		"annotations: [
			      [int,int,int,...,int],
			      ...
		]
	},
	...
]
```
and here's ascii art explaining the 2D grid to 1D vector map:
```text
2D:
[[pos-000, pos-001, pos-002, pos-003, ... , pos-031], (32 columns total)
 [pos-032, pos-033, pos-034, pos-035, ... , pos-063],
 [pos-064, pos-065, pos-066, pos-067, ... , pos-095],
 ... (18 rows total)
 [pos-544, pos-545, pos-546, pos-547, ... , pos-575]]

--- analogous to ---

1D:
[pos-000, pos-001, pos-002, pos-003, ... (576 positions total) , pos-572, pos-573, pos-547, pos-575]
```

The first row is a header. Every other row has a YouTube ID for the video and a list of pixel positions describing the 4 corners of a bounding box for each second. Here's the structure (whitespace added for clarity):
```csv
    ytid   , second1 ,         ,         ,         , second2 , , , ,second3, , , ,..., second9 ,         ,         ,
-----------,(row,col),(row,col),(row,col),(row,col),...                           ...,(row,col),(row,col),(row,col),(row,col)
...
```

## Separate Streams Captioning
Part 3 of this project is creating a new dataset. Videos contain 2 data streams - the auditory stream and the visual stream. Captioning sets like MSR-VTT ask annotators to watch a video containing both data streams. We hypothesize that humans will caption these videos mostly based on the visual stream (sort-of as an unknown bias). Thus, we want to caption each data stream separately. We created 2 Amazon Mechanical Turk interfaces to caption both data streams. Our goal is to caption 5000 videos spanning MSR-VTT and AVE. Each video should contain 2 audio captions and 2 video captions (totaling 4 captions). We are hoping these separate captions can provide insight as to how auditory information functions with visual information in general scene-understanding.

There are 2 interfaces for this task. Each interface should be a separate project. The audio-only interface displays the un-muted video with a black overlay covering the video. The video-only interface displays the muted video with no overlay. Both interfaces take the same `.csv` input file.

1. steps
1. ...
1. ...

Our Audio-Video Separate Streams dataset is formatted in JSON as follows (similar to MSR-VTT):
```json
{
	"info": {
		"year": str,
	        "version": str,
	        "description": str,
		"contributor": str,
		"data_created": str
	},
	"data": {
		"id": int,
		"video_id": str,
	        "origin": int,
		"ytid": str,
		"start time": float,
		"end time": float,
		"split": str,
	        "audio": {
			"sen_id": str(int) + 'a',
			"video_id": str,
			"caption": str
		},
		"video": {
			"sen_id": str(int) + 'v',
			"video_id": str,
			"caption": str
		}
    	}
}
```

Python 3 is required. We use Python 3.6. Here are the lib dependencies:
```bash
pip install csv json # you need boto3 if you want to try and use the mturk api
```

Download links for everything:
```bash
wget -nc http://storage.googleapis.com/us_audioset/youtube_corpus/v1/csv/unbalanced_train_segments.csv
wget -nc https://raw.githubusercontent.com/audioset/ontology/master/ontology.json
wget -nc http://vrttest2017.oss-us-east-1.aliyuncs.com/data/videodatainfo_2017.json
```
