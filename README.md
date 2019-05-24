# IoT - Create Predictive Maintenance Models To Detect Equipment Breakdown Risks in Maximo

## Description
Instrumented, connected assets generate volumes of operational data - structured and unstructured - that can be used to identify risks if the organizations have analytic tools to convey this insight to personnel responsible for asset operations.

In this journey, we'll show how to build and apply custom machine learning models to identify risks and suggest proactive maintenance to avoid service disruption. These models will also estimate how long a mechanical asset can be used before needing maintenance/replacement.

We'll also show how to import the custom models into a Maximo instance. Maximo is a system used for managing assets and workflow processes. This can be used to increase efficiency by automating processes such as work orders, notifications, anomaly detection, etc.


## Included Components
* [Maximo](https://www.ibm.com/products/maximo)
* [Maximo APM - Predictive Maintenance Insights](https://www.ibm.com/us-en/marketplace/predictive-maintenance-insights)
* [Watson Studio](https://cloud.ibm.com/catalog/services/watson-studio)

## Application Workflow Diagram
<img src="https://i.imgur.com/HkLyYwX.png">

## Flow
1. Build custom machine learning model in Watson Studio, and export custom model as python package
2. Publish sensor data from field assets to Maximo
3. Periodically pull asset data from Maximo to Python server, and generate predictions based on packaged custom model
4. Publish predicted "Remaining Useful Life" value to Maximo
5. (Within Maximo) Create "automation scripts" to execute when "Remaining Useful Life" is updated. If RUL below X number of days, create a work order to have asset replaced / updated.

# Steps
1. [Deploy Cloud Services](#1-deploy-cloud-services)
2. [Create Custom Model](#2-create-custom-model)
3. [Register Assets in Maximo](#3-register-assets-in-maximo)
4. [Create Automation Scripts](#4-create-automation-scripts)

## Install Prerequisites:
### Python
```
# OS X
brew install python python-pip

# Linux
apt-get install python python-pip

# Install python packages
pip install pickle scikit-learn flask
```


### 1. Provision services via IBM Cloud and IBM Marketplace dashboards
Create the following services
* [Maximo](https://www.ibm.com/products/maximo)
* [Maximo APM - Predictive Maintenance Insights](https://www.ibm.com/us-en/marketplace/predictive-maintenance-insights)
* [Watson Studio](https://cloud.ibm.com/catalog/services/watson-studio)

### 2. Create Custom Model
Navigate to your provisioned Watson Studio instance
<img src="https://i.imgur.com/6UxdEOO.png">

Click "Get Started"

Create a project. Select "Standard" project, and give it a name.
<img src="https://i.imgur.com/8xEgqW9.png">

<!-- TODO Add images for each of these -->
Select your created project and click "New Notebook"

Now, you can create your own notebook for a custom use case, or use one of our predefined notebooks. Here, we'll upload our preconfigured "Advanced Regression" notebook, which can be seen in further detail in the [notebooks](notebooks/Advanced_Regression_Turbofan.ipynb) directory of this project.

We'll do so by selecting "From File" in the menu, then entering a name, and the path to the local notebook file within this repo.
<img src="https://i.imgur.com/9GND5LT.png">

In this notebook, we'll be exploring a public dataset supplied by NASA. This dataset was generated by collecting sensor data from Turbofan engines. Each Turbofan has 20 sensors, as well as a "Cycles" field that signals how long a Turbofan has been running.

<!-- We'll start the notebook by importing a few popular python machine-learning libraries such as scikit-learn, pandas and matplotlib.

Next, we'll load our data, and "clean" it by removing null values. -->

<!-- We can also -->
<!-- TODO elaborate on "cleaning" data -->

After going through each step in the notebook, we can then export our model for consumption by Maximo PMI. This can be done by following the steps in the notebook at [notebooks/PMI-Custom_Model_Development.ipynb](notebooks/PMI-Custom_Model_Development.ipynb). You'll need the credentials that were included in your Welcome letter for the Maximo PMI instance.

Once this has been completed, we'll see the custom model available in the Maximo APM View.

As an alternative, we can also export the model by leveraging the `pickle` python package. This can be done by adding the following command to the notebook

```
pickle.dump(model, open("model.joblib", 'wb'))
```

### 3. Register Assets in Maximo
Now that we have our custom model built, the next step is to register assets in Maximo. This can be done by navigating to your Maximo instance.
<img src="https://i.imgur.com/UX7cJli.png">

In the "Find Navigation Item" field, enter "Asset Templates".
<img src="https://i.imgur.com/HbOYJFB.png">


Click "New Asset Template" in the left hand menu.
<!-- <img src="https://i.imgur.com/955fNzr.png" style="width:50%;height:50%"> -->

Give the Asset Template an ID (We used "Turbofan" here). We can also provide asset type, description, and other optional fields. Be sure to also enter "Turbofan" or some identifier in the description so we can use that as a search query.

<img src="https://i.imgur.com/2MOKIoK.png">

Directly below, select "Add Row" in the "Meters" section.

<img src="https://i.imgur.com/mtyJc0E.png">

In this case each meter will correspond to a sensor value. For simplicity, we'll add a single meter first, which measures the engine pressure.

<img src="https://i.imgur.com/JGNN9N3.png">

Save the Asset Template. Next, click "Generate New Assets", and create one or more assets.
<p align="center">
  <img src="https://i.imgur.com/f4Ia4k8.png" width="50%" height="50%" >
</p>
Now, we'll update the sensor data in one of our assets. To do so, enter "Assets" in the "Find Navigation Item" field.
<p align="center">
<img src="https://i.imgur.com/VvGULEP.png" width="50%" height="50%" >
</p>

Next, we can search for our Asset(s) by entering "Turbofan" in the description.
<img src="https://i.imgur.com/8vFxe9X.png">

Select an Asset, scroll down to the "More Actions" field and select "Enter Meter Readings", and then a value for each Meter.
<img src="https://i.imgur.com/wgNa1yp.png">

<img src="https://i.imgur.com/xLjhsrC.png">

These meter readings can also be updated via an HTTP POST request, we'll demonstrate this functionality in the next step.

### 4. Create Automation Scripts
Now, we'll leverage a set of python scripts to periodically query our registered assets, and collect the latest meter data. As each asset's associated data is collected, we'll load our custom model generated from Step 2 to estimate the remaining useful life of the asset.

In this particular implementation, we'll collect the asset data via the Maximo API.

To do so, we'll need to authenticate with a base64 encoded set of our Maximo credentials. These credentials are then provided in the following HTTP request as the "maxauth" header. We can query the API to get all meter readings associated with an Asset by sending a GET request to the following endpoint like so.

```
curl -H "maxauth: <base64 user:pw>" -H "Content-Type: application/json" ${MAXIMO_URL}/oslc/os/mxasset?oslc.select=assetmeter,expectedlife&oslc.where=assetnum=${ASSET_ID}
```

The database query is embedded at the end of the Maximo endpoint:
The `oslc.select` value determines which asset attributes will be returned (meter readings in this case)
The `oslc.where` value allows for us to only select data that matches a specific condition (matching our asset id)

After receiving the Asset data, we can estimate the Remaining Useful Life for our asset by loading our previous model using the `pickle` library
```
classifier = pickle.loads(load('model.joblib'))
predictions = classifier.predict(meter_data)
```

Finally, we can update the value for remaining useful life (expectedlife) in the Maximo database with the following POST request
```
requests.post("/oslc/os/mxasset/" + restid,
              data={"expectedlife": predictions[0]})
```

The full Maximo asset query/update workflow can be executed by running the included `server.py` script like so
```
export TOKEN=$(echo ${MAXIMO_USER}:${MAXIMO_PASS} | base64)
python server.py ${MAXIMO_URL} ${TOKEN} ${ASSET_ID}
```

In conclusion, here we have demonstrated how to create assets in Maximo, query their associated meters, and pass the metered data through a custom machine learning model. This allows for organizations to predict of when their various assets may fail, and proactively take action beforehand.

## License
This code pattern is licensed under the Apache Software License, Version 2. Separate third-party code objects invoked within this code pattern are licensed by their respective providers pursuant to their own separate licenses. Contributions are subject to the [Developer Certificate of Origin, Version 1.1 (DCO)](https://developercertificate.org/) and the [Apache Software License, Version 2](https://www.apache.org/licenses/LICENSE-2.0.txt).

[Apache Software License (ASL) FAQ](https://www.apache.org/foundation/license-faq.html#WhatDoesItMEAN)

<!-- TODO, add pic -->

<!-- Enter a integer value in the dialog box. Next, we'll create an automation script that will run whenever this value is updated

### 4. Create Automation Scripts
Next we'll create an Automation script based on the Jython (JAVA/Python) scripting language.

In the "Find Navigation Item" field, enter "Automation Scripts".

In the left menu, select "Create Script with Object Launch Point". A "Launch Point" is essentially an event that'll trigger a script. In this case, our Launch Point will be a change in Object state.

Script works like so
```
value = mbo.getDouble("measurementvalue")
```


To see more about Launch Points, please visit the documentation [here](https://www.ibm.com/support/knowledgecenter/SSLKT6_7.6.0/com.ibm.mbs.doc/autoscript/t_create_autocript.html) -->