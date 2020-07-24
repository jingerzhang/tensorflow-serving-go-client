# tensorflow-serving-go-client


## proto branch

tensorflow   r2.2   https://github.com/tensorflow/tensorflow/tree/r2.2

tensorflow serving  r2.2  https://github.com/tensorflow/serving/tree/r2.2


```shell
git clone -b r2.2 https://github.com/tensorflow/serving.git

git clone -b r2.2 https://github.com/tensorflow/tensorflow.git
```



## fix import cycle

`tensorflow_serving/core/logging.proto`



```proto
syntax = "proto3";

package tensorflow.serving;

import "google/protobuf/wrappers.proto";
import "tensorflow_serving/config/logging_config.proto";

option cc_enable_arenas = true;

// Metadata logged along with the request logs.
message LogMetadata {
  ModelSpec model_spec = 1;
  SamplingConfig sampling_config = 2;
  // List of tags used to load the relevant MetaGraphDef from SavedModel.
  repeated string saved_model_tags = 3;
  // TODO(b/33279154): Add more metadata as mentioned in the bug.
}

// Metadata for an inference request such as the model name and version.
message ModelSpec {
  // Required servable name.
  string name = 1;

  // Optional choice of which version of the model to use.
  //
  // Recommended to be left unset in the common case. Should be specified only
  // when there is a strong version consistency requirement.
  //
  // When left unspecified, the system will serve the best available version.
  // This is typically the latest version, though during version transitions,
  // notably when serving on a fleet of instances, may be either the previous or
  // new version.
  oneof version_choice {
    // Use this specific version number.
    google.protobuf.Int64Value version = 2;

    // Use the version associated with the given label.
    string version_label = 4;
  }

  // A named signature to evaluate. If unspecified, the default signature will
  // be used.
  string signature_name = 3;
}


```

## gen proto

```shell
#!/usr/bin/env bash

# git clone -b r2.2 https://github.com/tensorflow/serving.git
# git clone -b r2.2 https://github.com/tensorflow/tensorflow.git


if [[ ! -d vendor  ]];then
  eval "mkdir -p vendor"
fi

eval "rm -rf vendor/tensorflow"
eval "rm -rf vendor/tensorflow_serving"
eval "rm -rf vendor/github.com/tensorflow"


PROTOC_OPTS='-I tensorflow -I serving --go_out=vendor'
PROTOC_GRPC_OPTS='-I tensorflow -I serving --go-grpc_out=vendor'

directory_list=(
	tensorflow/tensorflow/core/framework
	tensorflow/tensorflow/core/example
	tensorflow/tensorflow/core/lib/core
	tensorflow/tensorflow/core/protobuf
	tensorflow/tensorflow/stream_executor
	serving/tensorflow_serving/apis
	serving/tensorflow_serving/config
	serving/tensorflow_serving/util
	serving/tensorflow_serving/sources/storage_path
	serving/tensorflow_serving/core
)

eval "protoc --version"

for directory in "${directory_list[@]}"; do
	file_list=$(ls ${directory} | grep .proto)
	for file in ${file_list}; do
        echo "protoc $PROTOC_OPTS ${directory}/${file}"
        eval "protoc $PROTOC_OPTS ${directory}/${file}"
        echo "protoc $PROTOC_GRPC_OPTS ${directory}/${file}"
        eval "protoc $PROTOC_GRPC_OPTS ${directory}/${file}"
	done
done

```

## go client

```go
package main

import (
	"context"
	"encoding/json"
	"flag"
	"log"

	ts "github.com/tensorflow/tensorflow/tensorflow/go/core/framework/tensor_go_proto"
	shape "github.com/tensorflow/tensorflow/tensorflow/go/core/framework/tensor_shape_go_proto"
	tftype "github.com/tensorflow/tensorflow/tensorflow/go/core/framework/types_go_proto"

	"google.golang.org/grpc"

	tf "tensorflow_serving/apis"
)

func main() {
	servingAddress := flag.String("serving-address", "localhost:8500", "The tensorflow serving address")
	flag.Parse()

	request := &tf.PredictRequest{
		ModelSpec: &tf.ModelSpec{
			Name:          "half_plus_two",
			SignatureName: "serving_default",
		},
		Inputs: map[string]*ts.TensorProto{
			"x": &ts.TensorProto{
				Dtype:    tftype.DataType_DT_FLOAT,
				FloatVal: []float32{2.0, 3.0, 4.5},
				TensorShape: &shape.TensorShapeProto{
					Dim: []*shape.TensorShapeProto_Dim{
						&shape.TensorShapeProto_Dim{
							Size: 3,
						},
					},
				},
			},
		},
	}

	message, _ := json.MarshalIndent(request, "", "    ")

	log.Printf("request: %s", message)

	conn, err := grpc.Dial(*servingAddress, grpc.WithInsecure())
	if err != nil {
		log.Fatalf("Cannot connect to the grpc server: %v\n", err)
	}
	defer conn.Close()

	client := tf.NewPredictionServiceClient(conn)

	resp, err := client.Predict(context.Background(), request)
	if err != nil {
		log.Fatalln(err)
	}
	message, _ = json.MarshalIndent(resp, "", "    ")

	log.Printf("resp: %s", message)
}


```
