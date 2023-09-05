# ComfyUI prompt control

Nodes for convenient prompt editing. The aim is to make basic generations in ComfyUI completely prompt-controllable.

The generated schedules depend on ComfyUI's timestep range feature unless you use the `PCSplitSampling` node.

Due to recent ComfyUI code refactoring, if you get import errors, you must update your ComfyUI installation.

You need to have `lark` installed in your Python environment for parsing to work (If you reuse A1111's venv, it'll already be there)

The basic nodes should now be stable, though I won't make interface guarantees quite yet.

[This example workflow](workflows/example.json) implements a two-pass workflow illustrating most features.

## Note on how schedules work

ComfyUI does not use the step number to determine whether to apply conds; instead, it uses the sampler's timestep value which affected by the scheduler you're using. This means that when the sampler scheduler isn't linear, the schedules generated by prompt control will not be either.

Currently there doesn't seem to be a good way to change this.

You can try using the `PCSplitSampling` node to enable an alternative method of sampling.

## Syntax

Syntax is like A1111 for now, but only fractions are supported for steps.

```
a [large::0.1] [cat|dog:0.05] [<lora:somelora:0.5:0.6>::0.5]
[in a park:in space:0.4]
```
### Alternating

Alternating syntax is `[a|b:pct_steps]`, causing the prompt to alternate every `pct_steps`. `pct_steps` defaults to 0.1 if not specified. You can also have more than two options.

### Sequences

The syntax `[SEQ:a:N1:b:N2:c:N3]` is shorthand for `[a:[b:[c::N3]:N2]:N1]` ie. it switches from `a` to `b` to `c` to nothing at the specified points in sequence.

Might be useful with Jinja templating (see Experiments below for details). For example:
```
[SEQ<% for x in steps(0.1, 0.9, 0.1) %>:<lora:test:<= sin(x*pi) + 0.1 =>>:<= x =><% endfor %>]
```
generates a LoRA schedule based on a sinewave

### Tag selection
Instead of step percentages, you can use a *tag* to select part of an input:
```
a large [dog:cat<lora:catlora:0.5>:SECOND_PASS]
```
You can then use the `tags` parameter in the `FilterSchedule` node to filter the prompt. If the tag matches any tag `tags` (comma-separated), the second option is returned (`cat`, in this case, with the LoRA). Otherwise, the first option is chosen (`dog`, without LoRA).

the values in `tags` are case-insensitive, but the tags in the input **must** be uppercase A-Z and underscores only, or they won't be recognized. That is, `[dog:cat:hr]` will not work.

For example, a prompt
```
a [black:blue:X] [cat:dog:Y] [walking:running:Z] in space
```
with `tags` `x,z` would result in the prompt `a blue cat running in space`

### Other syntax:

- `<emb:xyz>` is alternative syntax for `embedding:xyz` to work around a syntax conflict with `[embedding:xyz:0.5]` which is parsed as a schedule that switches from `embedding` to `xyz`.

- The keyword `BREAK` causes the prompt to be tokenized in separate chunks, which results in each chunk being individually padded to the text encoder's maximum token length.

- `AND` can be used to combine prompts. You can also use a weight at the end. It does a weighted sum of each prompt,
```
cat :1 AND dog :2
```
The weight defaults to 1 and are normalized so that `a:2 AND b:2` is equal to `a AND b`. `AND` is processed after schedule parsing, so you can change the weight mid-prompt: `cat:[1:2:0.5] AND dog`

## Schedulable LoRAs
The `ScheduleToModel` node patches a model such that when sampling, it'll switch LoRAs between steps. You can apply the LoRA's effect separately to CLIP conditioning and the unet (model)

For me this seems to be quite slow without the --highvram switch because ComfyUI will shuffle things between the CPU and GPU. YMMV. When things stay on the GPU, it's quite fast.

## AITemplate support
LoRA scheduling supports AITemplate.

Due to sampler patching, your AITemplate nodes must be cloned to a directory called `AIT` under `custom_nodes` or the hijack won't find it.

AITemplate sometimes breaks because ComfyUI creates batches larger than what AITemplate can handle. You can apply [this patch](0001-Limit-batch-chunk-size-to-2.patch) to ComfyUI to make AITemplate more reliable. The patch is a hack though, and may slightly deoptimize non-AIT sampling

## Advanced CLIP encoding

If you have [Advanced CLIP Encoding nodes](https://github.com/BlenderNeko/ComfyUI_ADV_CLIP_emb/tree/master) cloned into your `custom_nodes`, you can use the syntax `STYLE:weight_interpretation:normalization` at the start of a prompt to affect how prompts are interpreted.

The style can be specified separately for each AND:ed prompt, but the first prompt is special; later prompts will "inherit" it as default. For example:

```
STYLE:A1111 a (red:1.1) cat with (brown:0.9) spots and a long tail AND an (old:0.5) dog AND a (green:1.4) (balloon:1.1)
```
will interpret everything as A1111, but
```
a (red:1.1) cat with (brown:0.9) spots and a long tail AND STYLE:A1111 an (old:0.5) dog AND a (green:1.4) (balloon:1.1)
```
Will interpret the first one using the default ComfyUI behaviour, the second prompt with A1111 and the last prompt with the default again

For things (ie. the code imports) to work, the nodes must be cloned in a directory named exactly `ComfyUI_ADV_CLIP_emb`.

## Nodes

### PromptToSchedule
Parses a schedule from a text prompt. A schedule is essentially an array of `(valid_until, prompt)` pairs that the other nodes can use.

### FilterSchedule
Filters a schedule according to its parameters, removing any *changes* that do not occur within `[start, end)` as well as doing tag filtering. Always returns at least the last prompt in the schedule if everything would otherwise be filtered.

`start=0, end=0` returns the prompt at the start and `start=1.0, end=1.0` returns the prompt at the end.

### ScheduleToCond
Produces a combined conditioning for the appropriate timesteps. From a schedule. Also applies LoRAs to the CLIP model according to the schedule.

### ScheduleToModel
Produces a model that'll cause the sampler to reapply LoRAs at specific steps according to the schedule.

This depends on a callback handled by a monkeypatch of the ComfyUI sampler function, so it might not work with custom samplers, but it shouldn't interfere with them either.

### PCSplitSampling
Causes sampling to be split into multiple sampler calls instead of relying on timesteps for scheduling. This makes the schedules more accurate, but seems to cause weird behaviour with SDE samplers. (Upstream bug?)

### JinjaRender
Renders a String with Jinja2. See below for details

## Older nodes

- `EditableCLIPEncode`: A combination of `PromptToSchedule` and `ScheduleToCond`
- `LoRAScheduler`: A combination of `PromptToSchedule`, `FilterSchedule` and `ScheduleToModel`

## Utility nodes
### StringConcat
Concatenates the input strings into one string. All inputs default to the empty string if not specified

### ConditioningCutoff
Removes conditionings from the input whose timestep range ends before the cutoff and extends the remaining conds to cover the missing part. For example, set the cutoff to 1.0 to only leave the last prompt. This can be useful for HR passes.

# Experiments

## Cutoff node integration

If you have [ComfyUI Cutoff](https://github.com/BlenderNeko/ComfyUI_Cutoff) cloned into your `custom_nodes`, you can use the `CUT` keyword to use cutoff functionality

Currently, the syntax is:
```
a group of animals, white cat, brown dog CUT white cat; white CUT brown dog;brown;0.5;1.0;1.0;_`
```
the parameters in the `CUT` section are `region_text;target_text;weight;strict_mask;start_from_masked;padding_token` of which only the first two are required.
If `strict_mask`, `start_from_masked` or `padding_token` are specified in more than one section, the last one takes effect for the whole prompt

The syntax is likely to change at some point.

## Prompt interpolation

`a red [INT:dog:cat:0.2,0.8:0.05]` will attempt to interpolate the tensors for `a red dog` and `a red cat` between the specified range in as many steps of 0.05 as will fit.

## Jinja2
You can use the `JinjaRender` node to evaluate a string as a Jinja2 template. Note, however, that because ComfyUI's frontend uses `{}` for syntax, There are the following modifications to Jinja syntax:

- `{% %}` becomes `<% %>`
- `{{ }}` becomes `<= =>`
- `{# #}` becomes `<# #>`

Jinja stuff is experimental.

### Functions in Jinja templates

The following functions and constants are available:

- `pi`
- `min`, `max`, `clamp(minimum, value, maximum)`,
- `abs`, `round`, `ceil`, `floor`
- `sqrt` `sin`, `cos`, `tan`, `asin`, `acos`, `atan`. These functions are rounded to two decimals


In addition, a special `steps` function exists.

The `steps` function will generate a list of steps for iterating. 

You can call it either as `steps(end)`, `steps(end, step=0.1)` or `steps(start, end, step)`. `step` is an optional parameter that defaults to `0.1`. It'll return steps *inclusive* of start and end as long as step doesn't go past the end. 

The second form is equivalent to `steps(step, end, step)`. i.e. it starts at the first step.

# TODO & BUGS

The loaders can mostly reproduce the output from using `LoraLoader`.

More advanced workflows might explode horribly.

- `PCSplitSampling` with SDE schedulers overrides the noise sampling behaviour so that each split segment doesn't add crazy amounts of noise to the result
- Split sampling may have weird behaviour if your step percentages go below 1 step.
- Interpolation is probably buggy and will likely change behaviour whenever code gets refactored.
- The advanced prompt encoding integration seems to work, but I haven't verified if it gives the same results as the actual nodes.
- If execution is interrupted and LoRA scheduling is used, your models might be left in an undefined state until you restart ComfyUI
- Needs better syntax. A1111 is familiar, but not very good
- More advanced LoRA weight scheduling
