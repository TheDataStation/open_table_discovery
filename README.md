# Solo: Data Discovery Using Natural Language Questions Via A Self-Supervised Approach
This is the repository for the [paper](https://dl.acm.org/doi/10.1145/3626756) published at Sigmod 2024. Solo is a self-supervised data discovery system that finds tables among a large collection given natural language questions. It automatically generates training dataset from the target table collection and then trains the relevance model. Check out the [demo](https://drive.google.com/file/d/1tJzMa_rYH2O6TDELXQ6VAaiXsQx5_D-g/view?usp=drive_link) to see how to use Solo. The system consists of two stages:

Offline stage 

   First, tables in csv formats are converted to vectors and indexed. Second, during training, SQLs are automatically sampled from the table collection and translated to questions. Then training questions and tables are automatically collected to train the system.     

Online stage

   A user asks a question and the system returns top 5 tables that most likely answers the question.

To try the system, follow the following steps,

## 1. Install
We provide two options to install the system: install from repository (section 1.1) or load from docker image (section 1.2).

System requirements: GPU that supports CUDA 11.0 (above)

If possible, use solid state drive (SSD). Disk storage should be more than 300 G if you want to try all data released. 

### 1.1. Install from repository
In this option, the OS must be Ubuntu 18.04(or above)

a) Make sure Conda is installed

   Type "conda" in terminal, if "command not found", install Conda from https://docs.conda.io/en/latest/

b) Make sure the repository directory "solo" in an empty work directory <work_dir>
 
   A lot of data and other code will be downloaded to <work_dir>. If <work_dir> contains other child directories, there may be some collisions. After <work_dir> is created, move the "solo" directory in <work_dir> 
   
c) Create a new session using *tmux* or *screen*
   ```   bash
   tmux new -s [session name] or screen -S [session name] 
   ```
   It takes time to run the following scripts, so it is better to create a session.


d) Go to the "solo" directory and run the following script in the new session.
   ```   bash
   ./setup.sh
   conda activate s2ld
   ```

### 1.2. Load from docker image

a) Install NVIDIA Container Toolkit
   
   Checkout https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
   
   You can try step b) and c) first if you are not sure it is installer or not 

b) Load image
   ```   bash
   wget https://storage.googleapis.com/open_data_123/solo_docker.tar
   docker load -i solo_docker.tar
   ```
c) Start container
   ```   bash
   docker run --network host --name solo_app --gpus all -it -d solo_docker
   ```
   You only need to do this step once because the container (solo_app) is running in background after started

   Use the command "docker ps" to double check the container "solo_app" is running.

   If you want to restart container, stop (this will undo all changes, e.g. models and data downloaded) and remove it first by
   ```   bash
   docker stop solo_app
   docker rm solo_app
   ```
   Then you can start container again

d) Connect to container
   ```   bash
   docker exec -it --user cc -w /home/cc/solo_work/solo solo_app bash
   ```
   You do this step every time you want to use the system 

e) Update code
   In docker container
   ```   bash
   git pull
   ```

## 2. Model and Data
a) Download models
   ```   bash
   ./get_models.sh
   ```
b) Download data

   There are 3 datasets, "fetaqa", "nq_tables" and "chicago_open". 
   "chicago_open" is recommended to try.
   To download, run
   ```   bash
   ./get_data.sh <dataset>
   ```
   Each dataset is corresponding to a directory "<work_dir>/data/<dataset>". 

## 3. Index new datasets
   To index a new dataset (table collection), create 2 diretories,
    
   a) <work_dir>/data/\<dataset\>
   
   b) <work_dir>/data/\<dataset\>/tables_csv 
   
   We expect each table is in csv format with the filename denoting the title (caption) 
   and all csv files are in "<work_dir>/data/\<dataset\>/tables_csv" or 
   its offspring directories to allow same file names. 
   Then run the script
   ```   bash
   ./index.sh <dataset>
   ```
   You can also reindex these 3 datasets by running the script.

   Some important tips:

   **Customize table ID**:
   The system automatically assigns an ID for each table (the "tableId" field in each row of <work_dir>/data/\<dataset\>/tables/tables.jsonl). If you want to use your own table ID, you can follow the format <table_caption>\_SEP\_<table_ID> for the name of each csv file. It is your responsibilty to make sure table ID is unique, e.g. a GUID. You can customize the table ID for a subset of the tables and let the system assign others . For more details, read the code in the "table_from_csv.py" file.

   **Discovery in more fine-granularity**:
   You can split a table by row into multiple small tables so that you can locate a small region in the original table given a relevant query. Make sure to use the same caption for each small table. You can put these small tables in seperate subfolders if using same file names or just together if you customize the table ID

   **Sample rows for big tables**:
   If tables contain many rows, it can take too much disk space and the system will also be much slower. One solution is to sample K rows for those big tables. You can try K = 100 or other to keep a tradeoff among speed, space and search quality.

## 4. Train
   We provide pretrained models for "fetaqa" and "nq_talbes" and 
   you can ignore this if you just want to try table discovery (section 6).  
    
   The default batch size is 4, if the GPU memory is less than 24 G, use a smaller value (one by one) by updating "train_batch_size" in file "system.config". 
   
   By default, Incremental Training each time generates a dataset with 1,500 questions.
   
   If you want to retrain on "fetaqa" or "nq_tables", download the data first by "./get_data.sh". 
   
   If you want to train on other dataset, index the table collection first (section 3). 

   To train the relevance model, run
   ```   bash
   ./train.sh <dataset>
   ```
   After training, the best model will be deployed automatically to "<work_dir>/models/\<dataset\>" 

## 5. Test
   This is used to evaluate the retrieval accuracy of "fetaqa" and "nq_tables"
   ```   bash
   ./test.sh <dataset>
   ```
   We have pretrained models for "fetaqa" , "nq_tables" and "chicago_open" in 
   "<work_dir>/models/\<dataset\>" with file name "\<dataset\>_relevance.pt". 
   If you retrain "fetaqa" or "nq_tables", "<work_dir>/models/\<dataset\>" will have a best model for each training. 
   The script always loads the recent model (by create time), 
   so if you want to use the pretrained models, move the other model in some other directory.
    
## 6. Interactive Web application 
   The system provides a web interface.
   The user inputs a question and then top 5 tables are returned and displayed. 
   To try the application, follow the steps,

### 6.1. Start web server 
   run the script with the dataset "chicago_open"
   ```   bash
   ./run_server.sh chicago_open
   ```
   If the script is run a local machine where you can use a browser on it, 
   open "http://127.0.0.1:5000" and then go to section 6.3
   
### 6.2. Client setting 
   If the server is on a remote machine, do port routing on the client machine by runing the following script,
   ```   bash
   ssh -N -f -L 127.0.0.1:5000:127.0.0.1:5000 <user>@<remote server>
   ```
   Then on client, open "http://127.0.0.1:5000".
    
### 6.3. Try demo 
   Type a question to try, e.g.
   
   What are the business hours of Employment Resource Center at Howard?



