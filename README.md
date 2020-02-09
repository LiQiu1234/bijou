# bijou

A lightweight freamwork based on [fastai course](https://course.fast.ai) for training pytorch models conveniently. In particular, it is compatible with datasets and models of [pytorch_geometric](https://github.com/rusty1s/pytorch_geometric) and [DGL](https://docs.dgl.ai/en/latest/) for [Graph Neural Networks](https://arxiv.org/pdf/1812.08434.pdf).

## Features
- Compatible with PyG and DGL for GNN
  - Graph level learning: It is compatible with [pytorch_geometric](https://github.com/rusty1s/pytorch_geometric) and [DGL](https://docs.dgl.ai/en/latest/) for Graph Neural Networks of graph classification and other graph level learning.
  - Node level learning: It can be used in node classification or other node level learning with dataset of single [pytorch_geometric Data](https://pytorch-geometric.readthedocs.io/en/latest/modules/data.html) or [DGLGraph](https://docs.dgl.ai/en/latest/api/python/graph.html).
- Easy to Use
    - It likes [FastAI](https://docs.fast.ai) but far more lightweight. 

## Install

- `pip install bijou`

### Dependencies

  - Pytorch
  - Matplotlib
  - Numpy
  - tqdm
  - Networkx
  - torch-geometric   (Optional)
  - dgl               (Optional)

## Using

See following examples, and more examples are [here](https://github.com/hitlic/bijou/tree/master/examples).

## Examples

### a. MNIST classification

```python
import torch.nn as nn, torch.nn.functional as F, torch.optim as optim
from bijou.learner import Learner
from bijou.data import Dataset, DataLoader, DataBunch
from bijou.metrics import accuracy
from bijou.datasets import mnist
import matplotlib.pyplot as plt

# 1. dataset
x_train, y_train, x_valid, y_valid, x_test, y_test = mnist()
train_ds, valid_ds, test_ds = Dataset(x_train, y_train), Dataset(x_valid, y_valid), Dataset(x_test, y_test)
train_dl = DataLoader(train_ds, batch_size=128, shuffle=True)
valid_dl = DataLoader(valid_ds, batch_size=128)
test_dl = DataLoader(test_ds, batch_size=128)
# train_dl, valid_dl, test_dl = DataLoader.loaders(train_ds, valid_ds, test_ds, 128)
train_db = DataBunch(train_dl, valid_dl)

# 2. model and optimizer
in_dim = train_db.train_ds.x.shape[1]
out_dim = y_train.max().item()+1
model = nn.Sequential(nn.Linear(in_dim, 64), nn.ReLU(), nn.Linear(64, out_dim))
opt = optim.SGD(model.parameters(), lr=0.35)

# 3. learner
loss_func = F.cross_entropy
learner = Learner(model, opt, loss_func, train_db, metrics=[accuracy])

# 4. fit
learner.fit(10)

# 5. test
learner.test(valid_dl)

# 6. predict
pred = learner.predict(x_valid)
print(pred.size())

# 7.  plot
learner.recorder.plot_metrics()
plt.show()
```

### b. Graph Classification with PyG

NOTE: Performance of this GNN model's is not good, as the dataset is highly unbalanced.

```python
import torch, torch.nn as nn, torch.nn.functional as F, torch.optim as optim
from torch_geometric.nn import global_max_pool, TopKPooling, GCNConv
from bijou.learner import Learner
from bijou.datasets import pyg_yoochoose_10k
from bijou.data import DataBunch, PyGDataLoader
from bijou.metrics import accuracy
from examples.pyg_dataset import YooChooseBinaryDataset
import matplotlib.pyplot as plt

# 1. dataset
dataset = YooChooseBinaryDataset(root=pyg_yoochoose_10k()).shuffle()
train_ds, val_ds, test_ds = dataset[:8000], dataset[8000:9000], dataset[9000:]
train_dl = PyGDataLoader(train_ds, batch_size=64, shuffle=True)
val_dl = PyGDataLoader(val_ds, batch_size=64)
test_dl = PyGDataLoader(test_ds, batch_size=64)
# train_dl, val_dl, test_dl = PyGDataLoader.loaders(train_ds, val_ds, test_ds, 64)
train_db = DataBunch(train_dl, val_dl)

# 2. mode and optimizer
class Model(nn.Module):
    def __init__(self, feature_dim, class_num, embed_dim=64, gcn_dims=(32, 32), dense_dim=64):
        super().__init__()
        self.embedding = torch.nn.Embedding(num_embeddings=feature_dim, embedding_dim=embed_dim)
        self.gcns = nn.ModuleList()
        in_dim = embed_dim
        for dim in gcn_dims:
            self.gcns.append(GCNConv(in_dim, dim))
            in_dim = dim
        self.graph_pooling = TopKPooling(gcn_dims[-1], ratio=0.8)
        self.dense = nn.Linear(gcn_dims[-1], dense_dim)
        self.out = nn.Linear(dense_dim, class_num)

    def forward(self, data):
        x, edge_index, batch = data.x, data.edge_index, data.batch
        x = self.embedding(x)
        x = x.squeeze(1)
        for gcn in self.gcns:
            x = gcn(x, edge_index)
            x = F.relu(x)
        x, _, _, batch, _, _ = self.graph_pooling(x, edge_index, None, batch)
        x = global_max_pool(x, batch)
        outputs = self.dense(x)
        outputs = F.relu(outputs)
        outputs = self.out(outputs)
        return outputs

model = Model(dataset.item_num, 2)
opt = optim.SGD(model.parameters(), lr=0.5)

# 3. learner
learner = Learner(model, opt, F.cross_entropy, train_db, metrics=[accuracy])

# 4. fit
learner.fit(3)

# 5. test
learner.test(test_dl)

# 6. predict
pred = learner.predict(test_dl)
print(pred.size())

# 7. plot
learner.recorder.plot_metrics()
plt.show()
```

### c. Node Classification with PyG

```python
from torch_geometric.datasets import Planetoid
import torch.nn as nn, torch.nn.functional as F, torch.optim as optim
from torch_geometric.nn import GCNConv
from bijou.data import PyGGraphLoader, DataBunch
from bijou.learner import Learner
from bijou.metrics import masked_cross_entropy, masked_accuracy
from bijou.datasets import pyg_cora
import matplotlib.pyplot as plt

# 1. dataset
dataset = Planetoid(root=pyg_cora(), name='Cora')
train_dl = PyGGraphLoader(dataset, 'train')
val_dl = PyGGraphLoader(dataset, 'val')
test_dl = PyGGraphLoader(dataset, 'test')
# train_dl, val_dl, test_dl = PyGGraphLoader.loaders(dataset)
data = DataBunch(train_dl, val_dl)

# 2. model and optimizer
class Model(nn.Module):
    def __init__(self, feature_num, class_num):
        super().__init__()
        self.conv1 = GCNConv(feature_num, 16)
        self.conv2 = GCNConv(16, class_num)

    def forward(self, data):
        x, edge_index = data.x, data.edge_index
        x = self.conv1(x, edge_index)
        x = F.relu(x)
        x = self.conv2(x, edge_index)
        outputs = F.relu(x)
        return outputs

model = Model(dataset.num_node_features, dataset.num_classes)
opt = optim.SGD(model.parameters(), lr=0.5, weight_decay=0.01)

# 3. learner
learner = Learner(model, opt, masked_cross_entropy, data, metrics=[masked_accuracy])

# 4. fit
learner.fit(100)

# 5. test
learner.test(test_dl)

# 6. predict
pred = learner.predict(dataset[0])
print(pred.size())

# 7. plot
learner.recorder.plot_metrics()
plt.show()
```

### d. Graph Classification with DGL
```python

```

### e. Node Classification with DGL
```python

```