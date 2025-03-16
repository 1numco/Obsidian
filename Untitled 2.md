enum class TensorTypeBytes
{
	Bool,
	Int8,
	UInt8,
	...
}

class Tensor 
{
	TensorTypeBytes type;
	std::vector<int> dims;
	std::vector<uint8_t> data;
}

class ModelInput
{
	std::vector<std::pair<std::string, Tensor>> tensors; 
	
}
class ModelOutput
{
	Tensor tensor; 
};

struct EnclaveModel
{
	std::string purpose;
    TensorTypeBytes type;
    std::vector<int> dims;
    tflite::FlatBufferModel model;
    tflite::Interpreter interpreter;
};

struct Model
{
    void Infer (const ModelInput& input, ModelOutput& output);
    std::string purpose;
    TensorTypeBytes type;
    std::vector<int> dims;
};

struct ModelParams
{
	std::string name;
	char * buffer_data;
	size_t buffer_size;
};

struct Processor
{
    std::unique_ptr<Model> FindModel (const std::string& name);
    void LoadModels(const std::vector<ModelParams> model_files);
};