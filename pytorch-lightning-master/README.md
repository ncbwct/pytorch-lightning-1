<p align="center">
  <a href="https://williamfalcon.github.io/pytorch-lightning/">
    <img alt="" src="https://github.com/williamFalcon/pytorch-lightning/blob/master/imgs/lightning_logo.png" width="50">
  </a>
</p>
<h3 align="center">
  Pytorch Lightning
</h3>
<p align="center">
  The Keras for ML researchers using PyTorch. More control. Less boilerplate.    
</p>
<p align="center">
  <a href="https://badge.fury.io/py/pytorch-lightning"><img src="https://badge.fury.io/py/pytorch-lightning.svg" alt="PyPI version" height="18"></a>
<!--   <a href="https://travis-ci.org/williamFalcon/test-tube"><img src="https://travis-ci.org/williamFalcon/pytorch-lightning.svg?branch=master"></a> -->
  <a href="https://github.com/williamFalcon/pytorch-lightning/blob/master/LICENSE"><img src="https://img.shields.io/badge/License-MIT-yellow.svg"></a>
</p>   

```bash
pip install pytorch-lightning    
```

## Docs   
In progress. Documenting now!  

## Disclaimer 
This is a research tool I built for myself internally while doing my PhD. The API is not 100% production quality, but my hope is that by open-sourcing, we can all get it there (I don't have too much time nowadays to write production-level code).

## What is it?  
Keras is too abstract for researchers. Lightning makes it so you only have to define your model but still control all details of training if you need to.   

Pytorch   
<-- Lightning   
Your model.   

**Lightning will do the following for you:**   

1. Run the training loop.   
2. Run the validation loop.   
3. Run the testing loop.   
4. Early stopping.   
5. Learning rate annealing. 
6. Can train complex models like GANs or anything with multiple optimizers.
7. Weight checkpointing.   
8. Model saving.   
9. Model loading.   
10. Log training details (through test-tube).  
11. Run training on multiple GPUs (through test-tube).   
12. Run training on a GPU cluster managed by SLURM (through test-tube).   
13. Distribute memory-bound models on multiple GPUs.
14. Give your model hyperparameters parsed from the command line OR a JSON file.   
15. Run your model in a dev environment where nothing logs.      
  
## Usage
To use lightning do 2 things:  
1. [Define a trainer](https://github.com/williamFalcon/pytorch-lightning/blob/master/pytorch_lightning/trainer_main.py) (which will run ALL your models).   
2. [Define a model](https://github.com/williamFalcon/pytorch-lightning/blob/master/pytorch_lightning/models/sample_model_template/model_template.py).     

#### Basic trainer example  
See [this demo](https://github.com/williamFalcon/pytorch-lightning/blob/master/demo/fully_featured_trainer.py) for a more robust trainer example.   

```python
import os
import sys

from test_tube import HyperOptArgumentParser, Experiment
from pytorch_lightning.models.trainer import Trainer
from pytorch_lightning.utils.arg_parse import add_default_args
from pytorch_lightning.utils.pt_callbacks import EarlyStopping, ModelCheckpoint
from demo.example_model import ExampleModel


def main(hparams):
    """
    Main training routine specific for this project
    :param hparams:
    :return:
    """
    # init experiment
    exp = Experiment(
        name=hparams.tt_name,
        debug=hparams.debug,
        save_dir=hparams.tt_save_path,
        version=hparams.hpc_exp_number,
        autosave=False,
        description=hparams.tt_description
    )

    exp.argparse(hparams)
    exp.save()

    model_save_path = '{}/{}/{}'.format(hparams.model_save_path, exp.name, exp.version)

    # build model
    model = ExampleModel(hparams)

    # callbacks
    early_stop = EarlyStopping(monitor='val_acc', patience=3, mode='min', verbose=True)
    checkpoint = ModelCheckpoint(filepath=model_save_path, save_function=None, save_best_only=True, verbose=True, monitor='val_acc', mode='min')

    # configure trainer
    trainer = Trainer(experiment=exp, checkpoint_callback=checkpoint, early_stop_callback=early_stop)

    # train model
    trainer.fit(model)


if __name__ == '__main__':

    # use default args given by lightning
    root_dir = os.path.split(os.path.dirname(sys.modules['__main__'].__file__))[0]
    parent_parser = HyperOptArgumentParser(strategy='random_search', add_help=False)
    add_default_args(parent_parser, root_dir)

    # allow model to overwrite or extend args
    parser = ExampleModel.add_model_specific_args(parent_parser)
    hyperparams = parser.parse_args()

    # train model
    main(hyperparams)

```

#### Basic model example  
Here we only show the method signatures. It's up to you to define the content.   

```python
from torch import nn

class My_Model(RootModule):
    def __init__(self):
        # define model
        self.l1 = nn.Linear(200, 10)
    
    # ---------------
    # TRAINING
    def training_step(self, data_batch):
        x, y = data_batch
        y_hat = self.l1(x)
        loss = some_loss(y_hat)
        
        return loss_val, {'train_loss': loss}
    
    def validation_step(self, data_batch):
        x, y = data_batch
        y_hat = self.l1(x)
        loss = some_loss(y_hat)
        
        return loss_val, {'val_loss': loss}
 
     def validation_end(self, outputs):
        total_accs = []
        
        for output in outputs:
            total_accs.append(output['val_acc'].item())
        
        # return a dict
        return {'total_acc': np.mean(total_accs)}
     
     # ---------------
     # SAVING
     def get_save_dict(self):
        # lightning saves for you. Here's your chance to say what you want to save
        checkpoint = {'state_dict': self.state_dict()}

        return checkpoint

    def load_model_specific(self, checkpoint):
        # lightning loads for you. Here's your chance to say what you want to load
        self.load_state_dict(checkpoint['state_dict'])
    
    # ---------------
    # TRAINING CONFIG
    def configure_optimizers(self):
        # give lightning the list of optimizers you want to use.
        # lightning will call automatically
        optimizer = self.choose_optimizer('adam', self.parameters(), {'lr': self.hparams.learning_rate}, 'optimizer')
        return [optimizer]
    
    @property
    def tng_dataloader(self):
        return pytorch_dataloader('train')

    @property
    def val_dataloader(self):
        return pytorch_dataloader('val')

    @property
    def test_dataloader(self):
        return pytorch_dataloader('test')
        
    # ---------------
    # MODIFY YOUR COMMAND LINE ARGS
    @staticmethod
    def add_model_specific_args(parent_parser):    
        parser = HyperOptArgumentParser(strategy=parent_parser.strategy, parents=[parent_parser])
        parser.add_argument('--out_features', default=20)
        return parser
```


### Details    

#### Model definition   
| Name  | Description  |  Input |  Return |
|---|---|---|---|
| training_step  |  Called with a batch of data during training | data from your dataloaders  | tuple: scalar, dict  |
| validation_step  |  Called with a batch of data during validation  | data from your dataloaders  | tuple: scalar, dict |
| validation_end  |  Collate metrics from all validation steps |  outputs: array where each item is the output of a validation step | dict: for logging |
| get_save_dict  | called when your model needs to be saved (checkpoints, hpc save, etc...)  | None  |  dict to be saved | 

#### Model training   
| Name  | Description  |  Input |  Return |
|---|---|---|---|
| configure_optimizers  |  called during training setup | None | list: optimizers you want to use |
| tng_dataloader  |  called during training  | None  | pytorch dataloader |
| val_dataloader  |  called during validation  | None  | pytorch dataloader |
| test_dataloader  |  called during testing  | None  | pytorch dataloader |    
| add_model_specific_args  |  called with args you defined in your main. This lets you tailor args for each model and keep main the same  | argparse  | argparse |

#### Model Saving/Loading   
| Name  | Description  |  Input |  Return |
|---|---|---|---|
| get_save_dict  | called when your model needs to be saved (checkpoints, hpc save, etc...)  | None  |  dict to be saved |
|  load_model_specific |  called when loading a model | checkpoint: dict you created in get_save_dict  | dict: modified in whatever way you want  |
    
## Optional model hooks.   
Add these to the model whenever you want to configure training behavior.    


### Model lifecycle hooks
Use these hooks to customize functionality

| Method | Purpose  | Input  | Output  | Required  |
|---|---|---|---|---|
| on_batch_start()  | called right before the batch starts | - | -  | N  |
| on_batch_end()  | called right after the batch ends | - | -  | N  |
| on_epoch_start()  | called right before the epoch starts | - | -  | N  |
| on_epoch_end()  | called right afger the epoch ends | - | -  | N  |
| on_pre_performance_check()  | called right before the performance check starts | - | -  | N  |
| on_post_performance_check()  | called right after the batch starts | - | -  | N  |
