# Restaurant Sentiment Analysis Web Application

This application uses a machine learning model to classify given restaurant reviews based on their sentiment, being either positive (1) or negative (0).

## Starting the Application
To setup and run the application, do the following:
```bash
git clone https://github.com/remla2025-team10/operation
cd operation
docker-compose up
```

You can then access the application at [http://localhost:8080](http://localhost:8080). Here, you may enter a review into the textbox and query for a prediction.

### Additional startup configuration
There are also several environment variables which can be changed:
```bash
APP_IMAGE=ghcr.io/remla2025-team10/app-service:v0.4.11
MODEL_SERVICE_IMAGE=ghcr.io/remla2025-team10/model-service:v0.0.3
MODEL_SERVICE_PORT=3000
APP_PORT=8080
```

You can either change these manually, or when running `docker-compose up` like so:
```bash
 APP_PORT=5000 MODEL_SERVICE_PORT=3030 docker-compose up
 ```

## Relevant Files and Information
The application is structured in the following way:
- **Operation Repository** is the starting point of the application and contains the files required to run the application, as explained above
- **App Service Repository** holds the relevant frontend and backend code for the application
    - `app/models/model_handler.py` makes a post request to the model-service to get a sentiment prediction
    - `app/routes/__init__.py` defines the routes used by the backend application
    - `app/__init__.py` holds the overall backend application code
    - `app/templates/index.html` defines the frontend code of the application
- **Model Service Repository**
    - `model-service/download_models.py` is used to download the classifier (and vectorizer) model used for the prediction
    - `model-service/app.py` contains the Flask wrapper for the model, with an endpoint that can be accessed by the app-service to request a prediction for a given review
- **Lib-ml Repository**
    - `preprocess_sentiment_analysis/preprocess.py` is used to preprocess a given dataset by cleaning the text (removing non-alphabetical characters, converting all words to lowercase, removing stopwords, and Porter stemming). It saves the results into a processed CSV file.
- **Lib-version**
- **Model training**
    - `train.py` is used to train the restaurant sentiment analysis model.
    - `data/a1_RestaurantReviews_HistoricDump.tsv` is the training data that has been used to train the model.
    - `model/` contains the trained model files that is ready to use for the model-service.

## Repository Links
For convenience, we list links to the available repositories used in this project:
- [operation](https://github.com/remla2025-team10/operation)
- [app-service](https://github.com/remla2025-team10/app-service)
- [model-service](https://github.com/remla2025-team10/model-service)
- [model-training](https://github.com/remla2025-team10/model-training)
- [lib-version](https://github.com/remla2025-team10/lib-version)
- [lib-ml](https://github.com/remla2025-team10/lib-ml)

## Progress Log
### Assignment 1
#### What was done
All core components of the assignment have been implemented. 

- **Lib-ml** provides a preprocessing library for cleaning up the dataset text to be used in the sentiment analysis. 
- **Model-training** Trains the restaurant sentiment analysis model and publishes it to releases for access.
- **Model-service** applies preprocessing and uses the model to return a prediction.
- **Lib-version** [WRITE BRIEF SENTENCE HERE]. 
- **App-service** Build the frontend and backend for the project, call model-service api, and use lib-version util. 
- **Operation** builds up the containers for the model-service and app-service and hides model-service from the host (only accessible by app-service).

#### What is missing / needs improvement
[WRITE HERE IF ANYTHING COMES TO MIND]
