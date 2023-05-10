
# Install label studio

```
# install with anaconda
ssh -i /path/to/key ubuntu@xxx.xxx.xxx.x
conda create --name label-studio python=3.7
conda activate label-studio
pip install label-studio
```

# Start label studio

```
ssh -i /path/to/key ubuntu@xxx.xxx.xxx.x
tmux new -s label-studio
conda activate label-studio
label-studio start -db label_studio_postgresql --host http://xxx.xxx.xxx.x:8080
```

# Create accounts for label studio

- Sign up and create an account for Label Studio to start labeling data and setting up projects.

- For Community edition: Everyone with an account in Label Studio has access to the same functionality, they can access to all projects on tool.

# Set up the labeling interface

- All labeling activities in Label Studio occur in the context of a project. Hence, it is essential to set up the labeling interface and labeling configuration for your project.

- Steps:

  - Select a template from the available templates or customize one.

  - Add label names on new lines. Make sure the labels in your JSON file exactly match the labels in your configuration.

  - (Optional) Choose new colors for the labels.

- For OCR tasks you can follow this template: 
```xml
<View>
  <View style="display:flex;align-items:start;gap:8px;flex-direction:row"><Image name="image" value="$image" zoom="true" rotateControl="true" zoomControl="true"/><RectangleLabels name="label" toName="image" showInline="false">
    <Label value="signboard" background="green"/>
    <Label value="brand" background="blue"/>
    <Label value="address" background="red"/>
    <Label value="telephone" background="orange"/>
    <Label value="website" background="violet"/>
    <Label value="logo" background="brown"/>
    <Label value="text" background="Aquamarine"/>
  </RectangleLabels>
  
  <TextArea name="transcription" toName="image" editable="true" perRegion="true" required="false" maxSubmissions="1" rows="5" placeholder="Recognized Text" displayMode="region-list"/>

  </View>
</View>
```

# Create environment variable in .env 
```
LABEL_STUDIO_URL=xxx.xxx.xxx.x
API_KEY=dae462example21dc3278ab945cba5324a81bc0a #copy access token from http://xxx.xxx.xxx.x:8080/user/account
```

# Import data

### Prepare pre-annotated data with images input

- Label Studio JSON format for pre-annotations must contain two sections:

  - A `data` object which references the source of the data that the pre-annotations apply to. It could be an image URL or an image path in local.

  - A `predictions` array that contains the pre-annotation results.

    - `predictions.result`: an array of predictions, each contains 3 results dictionaries (Polygon, Labels, and TextArea) which have the same id and value.points referencing the same region.

    - The units the x, y of value.points are provided in percentages of overall image dimension.

- The JSON format for pre-annotations must match the labeling configuration (./label_config.xml) used for your data labeling project.

- Here is an example of input JSON file for OCR task:
```JSON
[{
   "data": {
      "ocr": "s3://data/upload/image1.jpg"
   },
   "predictions": [
      {
         "model_version": "best_ocr_model_1_final",
         "result": [
            {
               "original_width": 864,
               "original_height": 1296,
               "image_rotation": 0,
               "value": {
                  "points": [
                    [
                      10.701330108827086,
                      2.907225309961522
                    ],
                    [
                      51.20918984280532,
                      2.907225309961522
                    ],
                    [
                      51.20918984280532,
                      4.40359127832407
                    ],
                    [
                      10.701330108827086,
                      4.40359127832407
                    ]
                  ]
               },
               "id": "bb1",
               "from_name": "poly",
               "to_name": "image",
               "type": "polygon"
            },
            {
               "original_width": 864,
               "original_height": 1296,
               "image_rotation": 0,
               "value": {
                  "points": [
                    [
                      10.701330108827086,
                      2.907225309961522
                    ],
                    [
                      51.20918984280532,
                      2.907225309961522
                    ],
                    [
                      51.20918984280532,
                      4.40359127832407
                    ],
                    [
                      10.701330108827086,
                      4.40359127832407
                    ]
                  ],
                  "labels": [
                     "Text"
                  ]
               },
               "id": "bb1",
               "from_name": "label",
               "to_name": "image",
               "type": "labels"
            },
            {
               "original_width": 864,
               "original_height": 1296,
               "image_rotation": 0,
               "value": {
                  "points": [
                    [
                      10.701330108827086,
                      2.907225309961522
                    ],
                    [
                      51.20918984280532,
                      2.907225309961522
                    ],
                    [
                      51.20918984280532,
                      4.40359127832407
                    ],
                    [
                      10.701330108827086,
                      4.40359127832407
                    ]
                  ],
                  "text": [
                     "TOTAL"
                  ]
               },
               "id": "bb1",
               "from_name": "transcription",
               "to_name": "image",
               "type": "textarea"
            }
         ],
         "score": 0.89
      }
   ]
}]
```

- To start labeling on Label Studio, create projects using API or UI.

- Steps:

  - Create empty projects.

  - Set up labeling interface from ./label_config.xml

  - Import pre-annotated data.

  - (Optional) Adding source storage connections to a project makes private data viewable (just add, do not sync).

`import_data_to_label_studio.py`:

```python
import os
import json
from pathlib import Path
from dotenv import load_dotenv
load_dotenv()
import argparse
from label_studio_sdk import Client
import pandas as pd


if __name__=='__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('input_dir', type=str, help='path to import json folder')
    parser.add_argument('config_file', type=str, help='path to label configuration file')
    args = parser.parse_args()
    df = pd.read_csv(args.csv, dtype=str)
    LABEL_STUDIO_URL = os.getenv('LABEL_STUDIO_URL')
    ls = Client(url=LABEL_STUDIO_URL, api_key=os.getenv('API_KEY'))
    ls.check_connection()

    with open(args.config_file) as f:
        label_config = f.read()

    for pfile in Path(args.input_dir):
        title = pfile.stem # Specify the name of project
        data = json.load(open(pfile, 'r'))
        try:
            project = ls.start_project(
                title=title,
                label_config=label_config
            )
            project.import_tasks(data)
            
        except Exception as e:
            print(pfile.stem, e)
```

Run
`python import_data_to_label_studio.py input_dir /path/to/input_dir config_file /path/to/config_file`

# Label and annotate data
1. Open a project in Label Studio and optionally filter or sort the data.
2. Click `Label All Tasks` to start labeling.
3. Label the data:
  3.1. Select the label you want to apply to the region.
  3.2. Click the image and press your mouse to apply the label to the region.
  3.3. Click Submit to submit the completed annotation and move on to the next task.
5. Follow the project instructions for labeling and deciding whether to skip tasks.
6. Click the project name to return to the data manager.


# Export data

text file example:
```
test_001
test_002
test_003
```

`export_annotation_label_studio.py`:

```python
import os
from dotenv import load_dotenv
load_dotenv()
from label_studio_sdk import Client
import requests
import json
from pathlib import Path
import argparse
import pandas as pd

def list_available_export_format(LABEL_STUDIO_URL, prj_id):
    url = LABEL_STUDIO_URL + f'/api/projects/{prj_id}/export/formats'
    payload = {}
    files = {}
    headers = {'Authorization': 'Token ' + os.getenv('API_KEY'), 'Content-Type': 'application/json'}
    result = requests.request("GET", url, headers=headers, data=payload, files=files)
    valid_formats = []
    for format in result.json():
        if 'disabled' not in format or not format['disabled']:
            valid_formats.append(format['name'])
    return valid_formats

def export_file_in_the_specified_format(LABEL_STUDIO_URL, export_type, prj_id, prj_title, out_dir):
    # To export all tasks including tasks without annotations, add &download_all_tasks=true to the end of the below url
    url = LABEL_STUDIO_URL + f'/api/projects/{prj_id}/export?exportType={export_type}'
    payload = {}
    files = {}
    headers = {'Authorization': 'Token ' + os.getenv('API_KEY'), 'Content-Type': 'application/json'}
    r = requests.request("GET", url, headers=headers, data=payload, files=files)
    if r.json():
        export_file = out_dir / str(prj_title + '.json')
        with open(export_file, 'w') as f:
            json.dump(r.json(), f, indent=4)
        return export_file
    else:
        return None

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('out_dir', type=str, help='saved json directory', default='./')
    parser.add_argument('--txt', type=str, help='path to file containing project names')
    parser.add_argument('--export_format', type=str, help='format of export file', default='JSON')
    args = parser.parse_args()

    out_dir = Path(args.out_dir)
    out_dir.mkdir(parents=True, exist_ok=True)
    export_projects = []
    if args.txt:
      with open(args.txt) as f:
        export_projects = [line.strip('\n') for line in f.readlines()]
    export_format = args.export_format

    LABEL_STUDIO_URL = os.getenv('LABEL_STUDIO_URL')
    ls = Client(url=LABEL_STUDIO_URL, api_key=os.getenv('API_KEY'))
    
    for prj in ls.get_projects():
        title = prj.get_params()['title']
        if export_projects and title not in export_projects: continue
        try:
            project_id = prj.get_params()['id']
            valid_formats = list_available_export_format(LABEL_STUDIO_URL, project_id)
            assert input_format.upper() in valid_formats, f"export type is not correct, please choose one of these {valid_formats}"
            exported_file = export_file_in_the_specified_format(LABEL_STUDIO_URL, export_format, project_id, title, out_dir)
            print(f'{title} exported successfully: {exported_file}')
        
        except KeyError as e:
            print(f'{title} not in input list, no need exporting')
      

if __name__ == '__main__':
    main()
```

Run 
`python export_annotation_label_studio.py --out_dir /path/to/exported/json/folder --csv /path/to/csv/file`

 
