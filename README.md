# CNN-with-PyTorch:

A convolutional neural network (CNN) is a kind of neural network that extracts features from matrices of numeric values (often images) by convolving multiple filters over the matrix values to apply weights and identify patterns, such as edges, corners, and so on in an image. The numeric representations of these patterns are then passed to a fully-connected neural network layer to map the features to specific classes.

## Environment:

This project is developed and tested in Google Colab, which provides an excellent platform for running machine learning experiments with access to GPUs and TPUs. Below are the details of the environment and dependencies used:

- **Python Version**: Google Colab provides Python 3.7+
- **PyTorch Version**: 2.3.1
- **Torchvision Version**: 0.18.1
- **Torchaudio Version**: 2.3.1

### Requirements:

To ensure compatibility and reproducibility, the following versions of PyTorch and its related libraries are used:

```
torch==2.3.1
torchvision==0.18.1
torchaudio==2.3.1
```

## Below are the libraries used:

```
# Import PyTorch libraries
import torch
import torchvision
import torchvision.transforms as transforms
import torch.nn as nn
import torch.optim as optim
from torch.autograd import Variable
import torch.nn.functional as F

# Other libraries we'll use
import numpy as np
import os
import matplotlib.pyplot as plt
import matplotlib.image as mpimg
%matplotlib inline
```

### Uploading Local Folder to Google Colab:
To upload a local folder to Google Colab, follow these steps:

Download the zip file from Kaggle.
Zip the folder on your local PC.
Upload the zip file to Colab using files.upload().
Unzip the file in Colab using zipfile.ZipFile

```
import zipfile
import os
```

### Data Exploration:
To understand the dataset better, I display the first image from each category. Below is an example image showing the first image from three different shape categories: Square, Triangle, and Circle.

```
# Show the first image in each folder
fig = plt.figure(figsize=(8, 12))
i = 0
for sub_dir in os.listdir(data_path):
    i += 1
    img_file = os.listdir(os.path.join(data_path, sub_dir))[0]
    img_path = os.path.join(data_path, sub_dir, img_file)
    img = mpimg.imread(img_path)
    a = fig.add_subplot(1, len(classes), i)
    a.axis('off')
    imgplot = plt.imshow(img)
    a.set_title(img_file)
plt.show()

```
This code iterates through each subdirectory in the dataset, reads the first image file, and displays it in a matplotlib figure, providing a quick visual overview of the dataset.

![image](https://github.com/Tima-R/CNN-with-PyTorch/assets/116596345/d31f8d82-562b-4682-8233-1b773aeac102)


The images are labeled with their respective shapes and an identifier. This visualization helps us verify the contents and structure of the dataset, ensuring that the images are correctly categorized and loaded. Each subplot title shows the shape and its corresponding filename. This is useful for a quick visual inspection of the dataset.

### Load data:
I use the following function to load the dataset, creating training and testing data loaders.

```
# Function to ingest data using training and test loaders
def load_dataset(data_path):
    # Load all of the images
    transformation = transforms.Compose([
        # transform to tensors
        transforms.ToTensor(),
        # Normalize the pixel values (in R, G, and B channels)
        transforms.Normalize(mean=[0.5, 0.5, 0.5], std=[0.5, 0.5, 0.5])
    ])

    # Load all of the images, transforming them
    full_dataset = torchvision.datasets.ImageFolder(
        root=data_path,
        transform=transformation
    )


    # Split into training (70% and testing (30%) datasets)
    train_size = int(0.7 * len(full_dataset))
    test_size = len(full_dataset) - train_size
    train_dataset, test_dataset = torch.utils.data.random_split(full_dataset, [train_size, test_size])

    # define a loader for the training data we can iterate through in 50-image batches
    train_loader = torch.utils.data.DataLoader(
        train_dataset,
        batch_size=50,
        num_workers=0,
        shuffle=False
    )

    # define a loader for the testing data we can iterate through in 50-image batches
    test_loader = torch.utils.data.DataLoader(
        test_dataset,
        batch_size=50,
        num_workers=0,
        shuffle=False
    )

    return train_loader, test_loader


# Get the iterative dataloaders for test and training data
train_loader, test_loader = load_dataset(data_path)
print('Data loaders ready')

```

This function loads all images, applies transformations (converting to tensors and normalizing), splits the data into training (70%) and testing (30%) sets, and creates iterative data loaders for batch processing.


### Define the CNN:
A convolutional neural network (CNN) model using PyTorch is defined. The architecture includes three convolutional layers with ReLU activations and max pooling, followed by a dropout layer to prevent overfitting. The final fully connected layer maps the extracted features to class probabilities.

```
# Create a neural net class
class Net(nn.Module):
    # Constructor
    def __init__(self, num_classes=3):
        super(Net, self).__init__()
        
        # Our images are RGB, so input channels = 3. We'll apply 12 filters in the first convolutional layer
        self.conv1 = nn.Conv2d(in_channels=3, out_channels=12, kernel_size=3, stride=1, padding=1)
        
        # We'll apply max pooling with a kernel size of 2
        self.pool = nn.MaxPool2d(kernel_size=2)
        
        # A second convolutional layer takes 12 input channels, and generates 12 outputs
        self.conv2 = nn.Conv2d(in_channels=12, out_channels=12, kernel_size=3, stride=1, padding=1)
        
        # A third convolutional layer takes 12 inputs and generates 24 outputs
        self.conv3 = nn.Conv2d(in_channels=12, out_channels=24, kernel_size=3, stride=1, padding=1)
        
        # A drop layer deletes 20% of the features to help prevent overfitting
        self.drop = nn.Dropout2d(p=0.2)
        
        # Our 128x128 image tensors will be pooled twice with a kernel size of 2. 128/2/2 is 32.
        # So our feature tensors are now 32 x 32, and we've generated 24 of them
        # We need to flatten these and feed them to a fully-connected layer
        # to map them to  the probability for each class
        self.fc = nn.Linear(in_features=32 * 32 * 24, out_features=num_classes)

    def forward(self, x):
        # Use a relu activation function after layer 1 (convolution 1 and pool)
        x = F.relu(self.pool(self.conv1(x)))
      
        # Use a relu activation function after layer 2 (convolution 2 and pool)
        x = F.relu(self.pool(self.conv2(x)))
        
        # Select some features to drop after the 3rd convolution to prevent overfitting
        x = F.relu(self.drop(self.conv3(x)))
        
        # Only drop the features if this is a training pass
        x = F.dropout(x, training=self.training)
        
        # Flatten
        x = x.view(-1, 32 * 32 * 24)
        # Feed to fully-connected layer to predict class
        x = self.fc(x)
        # Return log_softmax tensor 
        return F.log_softmax(x, dim=1)
    
print("CNN model class defined!")

```










































