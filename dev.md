# Step1. Build docker
 ```
 # server
 docker build . -f Dockerfile/dockerfile.server -t wespeaker:latest --network host
 # client
 docker build . -f Dockerfile/dockerfile.client -t wespeaker_client:latest --network host
 ```

# Step2. Convert model to tensorrt engine
The tensorrt version should be consistent with the triton inference server docker. For example,
we use 8.2.3 in triton 22.03. Therefore, you may use tensorrt-22.03 docker to convert your model.

 ```
cd model_repo/speaker_model/
# Convert resnet model. We set the maximum sequence length as 3000, which is 30 seconds
# If your model only needs 10 seconds, you may simply reduce 3000 to 1000
# Similarly, we set 200 as the minimum length of model just because all our samples length are
# larger than 2 seconds.
# You may ask your algorithm engineer to confirm these numbers for your customized models.
trtexec --saveEngine=b1_b128_s3000_fp16.trt  --onnx=resnet/resnet_avg_model.onnx --minShapes=feats:1x200x80 --optShapes=feats:64x200x80 --maxShapes=feats:128x3000x80 --fp16

# Convert your int8 model by trtexec
# There is also one example onnx model with quantizers we inserted manually and did the PTQ.
trtexec --saveEngine=b1_b128_s3000_int8.trt  --onnx=resnet/resnet_avg_model_calibrate-max.onnx --minShapes=feats:1x200x80 --optShapes=feats:64x200x80 --maxShapes=feats:128x3000x80 --fp16 --int8
# int8 model will only accelerate on GPUs like T4, A30, A100, ...

# Edit the config.pbtxt under this directory to the generate engine name
```
 
# Step3. Start server
 ```
 docker run --gpus '"device=0"' -v $PWD/model_repo:/ws/model_repo -v $MODEL_DIR:/ws/models --shm-size=1g --ulimit memlock=-1 -p 8000:8000 -p 8001:8001 -p 8002:8002 --ulimit stack=67108864 -ti  wespeaker:latest
 tritonserver --model-repository=/ws/model_repo
 ```
 
# Step4. Start client
```
docker run -it -v $pwd:/ws -v $dataset:/dataset wespeaker_client

# example command
cd /ws/client/
python3 client.py --url=localhost:8001 --wavscp=/raid/dgxsa/slyne/wespeaker/examples/voxceleb/v2/data/vox1/wav.scp --output_directory=<to put the generated embeddings>

# The output direcotry will be something like:
# xvector_000.ark xvextor_000.scp xvector_001.scp .....

```

# Step5. Test scores
We use the same score function as wespeaker to test these embedding scores.
Please ask your algorithm engineer to test.
```
cat embeddings/xvector_*.scp > embeddings/xvector.scp
```
The function looks like:
```
echo "Apply cosine scoring ..."
config=conf/resnet.yaml
exp_dir=exp/resnet

mkdir -p embeddings/scores
trials_dir=data/vox1/trials
python -u wespeaker/bin/score.py \
--exp_dir ${exp_dir} \
--eval_scp_path /raid/dgxsa/slyne/wespeaker/runtime/server/x86_gpu/embeddings/xvector.scp \
--cal_mean True \
--cal_mean_dir ${exp_dir}/embeddings/vox2_dev \
--p_target 0.01 \
--c_miss 1 \
--c_fa 1 \
${trials_dir}/vox1_O_cleaned.kaldi ${trials_dir}/vox1_E_cleaned.kaldi ${trials_dir}/vox1_H_cleaned.kaldi \
2>&1 | tee /raid/dgxsa/slyne/wespeaker/runtime/server/x86_gpu/embeddings/scores/vox1_cos_result
```


# Step 6  Others
There's one ```export_onnx.py``` file. You may use this file to export your own onnx model.
I've added the quantization part. If there's no quantizers in your model, you may just ignore the
`--quantized` option.
Please put this file to wespeaker/bin/ when using this script.

Notice there's one option (--mean_vec) that allows you to put the mean vector into the generated onnx model.
If you don't need the put to minus the mean vector and do this by yourself or just like the `score.py` which can do it automatically. You can just skip it. Just a reminder, don't forget about this vector.
