训练ssg:
```
python train_ssg.py -i "data_folder" -b "batch_size" -e "number of epochs" -o "output folder"
```

预测:

```
python ssg_prediction.py -i "data_folder" -m "model_address" -th list_of_thresholds
```

评估:

```
python evaluate_set_ssg.py -i "prediction file"
```