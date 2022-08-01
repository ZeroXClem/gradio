# Gradio and Wandb Integration

Related spaces: https://huggingface.co/spaces/akhaliq/JoJoGAN
Tags: WANDB, SPACES
Contributed by Gradio team
Docs: image, Dropdown


## Introduction

In this Guide, we'll walk you through:

* Introduction of Gradio, and Hugging Face Spaces, and Wandb
* How to setup a Gradio demo using the Wandb integration for JoJoGAN
* How to contribute your own Gradio demos after tracking your experiments on wandb to the Wandb organization on Hugging Face

Here's an example of an model trained and experiments tracked on wandb, try out the JoJoGAN demo below.

<script type="module" src="https://gradio.s3-us-west-2.amazonaws.com/3.0.18/gradio.js"></script>
<gradio-app space="akhaliq/JoJoGAN"> </gradio-app>

## What is the Wandb?

Weights and Biases (W&B) allows data scientists and machine learning scientists to track their machine learning experiments at every stage, from training to production. Any metric can be aggregated over samples and shown in panels in a customizable and searchable dashboard, like below:

## What are Hugging Face Spaces & Gradio?

### Gradio

Gradio lets users demo their machine learning models as a web app all in python code. Gradio wraps a python function into a user inferface and the demos can be launched inside jupyter notebooks, colab notebooks, as well as embedded in your own website and hosted on Hugging Face Spaces for free.

Get started [here](https://gradio.app/getting_started)

### Hugging Face Spaces

Hugging Face Spaces is a free hosting option for Gradio demos. Spaces comes with 3 SDK options: Gradio, Streamlit and Static HTML demos. Spaces can be public or private and the workflow is similar to github repos. There are over 2000+ spaces currently on Hugging Face. Learn more about spaces [here](https://huggingface.co/spaces/launch).


## Setting up a Gradio Demo for JoJoGAN

Now, let's walk you through how to do this on your own. We'll make the assumption that you're new to W&B and Gradio for the purposes of this tutorial. 

Let's get started!


1. Create a W&B account

Follow these quick instructions to create your free account if you don’t have one already. It shouldn't take more than a couple minutes. Once you're done (or if you've already got an account), next, we'll run a quick colab. 


2. Open Colab Install Gradio and W&B


We'll be following along with the colab provided in the JoJoGAN repo with some minor modifications to use Wandb and Gradio more effectively. 

Install Gradio and Wandb at the top:

```!pip install gradio wandb```

3. Follow the instruction in colab to setup and try out a pretrained model

Code as follows: 


```
plt.rcParams['figure.dpi'] = 150
pretrained = 'arcane_multi' #@param ['art', 'arcane_multi', 'supergirl', 'arcane_jinx', 'arcane_caitlyn', 'jojo_yasuho', 'jojo', 'disney']
#@markdown Preserve color tries to preserve color of original image by limiting family of allowable transformations. Otherwise, the stylized image will inherit the colors of the reference images, leading to heavier stylizations.
preserve_color = False #@param{type:"boolean"}


if preserve_color:
    ckpt = f'{pretrained}_preserve_color.pt'
else:
    ckpt = f'{pretrained}.pt'


# load base version if preserve_color version not available
try:
    downloader.download_file(ckpt)
except:
    ckpt = f'{pretrained}.pt'
    downloader.download_file(ckpt)


ckpt = torch.load(os.path.join('models', ckpt), map_location=lambda storage, loc: storage)
generator.load_state_dict(ckpt["g"], strict=False)


#@title Generate results
n_sample =  5#@param {type:"number"}
seed = 3000 #@param {type:"number"}


torch.manual_seed(seed)
with torch.no_grad():
    generator.eval()
    z = torch.randn(n_sample, latent_dim, device=device)


    original_sample = original_generator([z], truncation=0.7, truncation_latent=mean_latent)
    sample = generator([z], truncation=0.7, truncation_latent=mean_latent)


    original_my_sample = original_generator(my_w, input_is_latent=True)
    my_sample = generator(my_w, input_is_latent=True)


# display reference images
if pretrained == 'arcane_multi':
    style_path = f'style_images_aligned/arcane_jinx.png'
else:   
    style_path = f'style_images_aligned/{pretrained}.png'
style_image = transform(Image.open(style_path)).unsqueeze(0).to(device)
face = transform(aligned_face).unsqueeze(0).to(device)


my_output = torch.cat([style_image, face, my_sample], 0)
display_image(utils.make_grid(my_output, normalize=True, range=(-1, 1)), title='My sample')


output = torch.cat([original_sample, sample], 0)
display_image(utils.make_grid(output, normalize=True, range=(-1, 1), nrow=n_sample), title='Random samples')
```

4. Add style images for fine-tuning

Next, you'll upload some of your own images for style. Upload those images in colab and add the names of the images as shown below:


```
Upload your own style images into the style_images folder and type it into the field in the following format without the directory name. Upload multiple style images to do multi-shot image translation
names = ['arcane_caitlyn.jpeg', 'arcane_jinx.jpeg', 'arcane_jayce.jpeg', 'arcane_viktor.jpeg'] #@param {type:"raw"}


targets = []
latents = []


for name in names:
    style_path = os.path.join('style_images', name)
    assert os.path.exists(style_path), f"{style_path} does not exist!"


    name = strip_path_extension(name)


    # crop and align the face
    style_aligned_path = os.path.join('style_images_aligned', f'{name}.png')
    if not os.path.exists(style_aligned_path):
        style_aligned = align_face(style_path)
        style_aligned.save(style_aligned_path)
    else:
        style_aligned = Image.open(style_aligned_path).convert('RGB')


    # GAN invert
    style_code_path = os.path.join('inversion_codes', f'{name}.pt')
    if not os.path.exists(style_code_path):
        latent = e4e_projection(style_aligned, style_code_path, device)
    else:
        latent = torch.load(style_code_path)['latent']


    targets.append(transform(style_aligned).to(device))
    latents.append(latent.to(device))


targets = torch.stack(targets, 0)
latents = torch.stack(latents, 0)


target_im = utils.make_grid(targets, normalize=True, range=(-1, 1))
display_image(target_im, title='Style References')
```


5. Finetune StyleGAN and W&B experiment tracking

This next step will open a W&B dashboard to track your experiments and a gradio panel showing pretrained models to choose from a drop down menu from a Gradio Demo hosted on Huggingface Spaces.

```
#@title Finetune StyleGAN
#@markdown alpha controls the strength of the style
alpha =  1.0 #@param {type:"slider", min:0, max:1, step:0.1}
alpha = 1-alpha


#@markdown Tries to preserve color of original image by limiting family of allowable transformations. Set to false if you want to transfer color from reference image. This also leads to heavier stylization
preserve_color = True #@param{type:"boolean"}
#@markdown Number of finetuning steps. Different style reference may require different iterations. Try 200~500 iterations.
num_iter = 200 #@param {type:"number"}
#@markdown Log training on wandb and interval for image logging
use_wandb = True #@param {type:"boolean"}
log_interval = 50 #@param {type:"number"}


samples = []
column_names = ["Referece (y)", "Style Code(w)", "Real Face Image(x)"]


if use_wandb:
    wandb.init(project="JoJoGAN")
    config = wandb.config
    config.num_iter = num_iter
    config.preserve_color = preserve_color
    wandb.log(
    {"Style reference": [wandb.Image(transforms.ToPILImage()(target_im))]},
    step=0)
    wandb.log({"Gradio panel": wandb.Html('''
<link rel="stylesheet" href="https://gradio.s3-us-west-2.amazonaws.com/2.6.2/static/bundle.css">
<div id="JoJoGAN-demo"></div>
<script src="https://gradio.s3-us-west-2.amazonaws.com/2.6.2/static/bundle.js"></script>
<script>
launchGradioFromSpaces("akhaliq/JoJoGAN", "#JoJoGAN-demo")
</script>
<style>
/* work around a weird bug */
.gradio_app .gradio_bg[theme=huggingface] .gradio_interface .input_dropdown .dropdown:hover .dropdown_menu {
    display: block;
}
</style>
''', inject=False)})


lpips_fn = lpips.LPIPS(net='vgg').to(device)


# reset generator
del generator
generator = deepcopy(original_generator)


g_optim = optim.Adam(generator.parameters(), lr=2e-3, betas=(0, 0.99))


# Which layers to swap for generating a family of plausible real images -> fake image
if preserve_color:
    id_swap = [7,9,11,15,16,17]
else:
    id_swap = list(range(7, generator.n_latent))


for idx in tqdm(range(num_iter)):
    if preserve_color:
        random_alpha = 0
    else:
        random_alpha = np.random.uniform(alpha, 1)
    mean_w = generator.get_latent(torch.randn([latents.size(0), latent_dim]).to(device)).unsqueeze(1).repeat(1, generator.n_latent, 1)
    in_latent = latents.clone()
    in_latent[:, id_swap] = alpha*latents[:, id_swap] + (1-alpha)*mean_w[:, id_swap]


    img = generator(in_latent, input_is_latent=True)
    loss = lpips_fn(F.interpolate(img, size=(256,256), mode='area'), F.interpolate(targets, size=(256,256), mode='area')).mean()
    
    if use_wandb:
        wandb.log({"loss": loss}, step=idx)
        if idx % log_interval == 0:
            generator.eval()
            my_sample = generator(my_w, input_is_latent=True)
            generator.train()
            my_sample = transforms.ToPILImage()(utils.make_grid(my_sample, normalize=True, range=(-1, 1)))
            wandb.log(
            {"Current stylization": [wandb.Image(my_sample)]},
            step=idx)
	    table_data = [
                wandb.Image(transforms.ToPILImage()(target_im)),
                wandb.Image(img),
                wandb.Image(my_sample),
            ]
            samples.append(table_data)


    g_optim.zero_grad()
    loss.backward()
    g_optim.step()


out_table = wandb.Table(data=samples, columns=column_names)
wandb.log({"Current Samples": out_table})
```



Using Web components, using the <gradio-app> tags allows anyone can embed Gradio demos on HF spaces directly into their blogs, websites, documentation, etc.:

```<gradio-app space="akhaliq/JoJoGAN"> </gradio-app>```

Meanwhile, adding a Gradio Demo to a W&B Report takes just a few extra lines of code: 


```
wandb.log({"Gradio panel": wandb.Html('''
<script type="module" src="https://gradio.s3-us-west-2.amazonaws.com/3.0.18/gradio.js"></script>
<gradio-app space="akhaliq/JoJoGAN"> </gradio-app>
)
```

## How to contribute Gradio demos on HF spaces on the Wandb organization

* Create an account on Hugging Face [here](https://huggingface.co/join).
* Add Gradio Demo under your username, see this [blog post](https://huggingface.co/blog/gradio-spaces) for setting up Gradio Demo on Hugging Face. 
* Request to join wandb organization [here](https://huggingface.co/wandb).
* Once approved transfer model from your username to Wandb organization