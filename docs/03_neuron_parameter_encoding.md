# Neuron Parameter Encoding — write_neuron_type Bit Field Layout

The `write_neuron_type` command programs the neuron_param_mem BRAM in each core. The 84-bit command word is constructed in `fpga_controller.py` and read by the IEP.

## Bit Field Layout (84 bits total)

```
Bit Range    Field                Width   Description
─────────────────────────────────────────────────────────────────
[83:78]      leak                 6       Leak rate (0-63). leak=63 with LIF gives no leak.
[77:72]      shift                6       Noise shift (signed in L6d+).
                                          shift=-17 (6'b101111) = no noise.
                                          Positive = left shift (more noise).
[71:70]      neuronModel          2       0=ANN, 1=counter, 2=LIF, 3=passthrough
[69:34]      threshold            36      Spike threshold (signed in L6d+)
[33:21]      boundary             13      (stopAddr+15)//16. URAM rows to process.
[20:17]      delay_value          4       Synaptic delay countdown (0-15 timesteps)
[16:14]      refractory_max       3       Refractory period (0-7 timesteps)
[13]         dual_synapse_en      1       Enable Group A/B synapse separation
[12:9]       shadow_uram_offset   4       Offset for shadow URAM in dual-synapse mode
[8]          soft_reset_en        1       0=hard reset (MP=0), 1=soft reset (MP=MP-threshold)
[7]          legacy_noise_en      1       1=2024 mode (35-bit MP), 0=L6d mode (32-bit MP)
[6:0]        (unused)             7       Reserved for future use
```

## Software Encoding (fpga_controller.py)

```python
def write_neuron_type(stopAddr, Threshold, neuronModel, shift, leak,
                      refractory_max=0, dual_synapse_en=0, delay_value=0,
                      shadow_uram_offset=0, legacy_noise_en=0,
                      coreID=0, simDump=False):
    command = ['0'] * 512
    command[-84:-78] = list(np.binary_repr(leak, 6))
    command[-78:-72] = list(np.binary_repr(shift % 64, 6))  # 6-bit twos complement
    command[-72:-70] = list(np.binary_repr(neuronModel, 2))
    command[-70:-34] = list(np.binary_repr(Threshold % (2**36), 36))
    command[-34:-21] = list(np.binary_repr((stopAddr+15)//16, 13))  # boundary
    command[-21:-17] = list(np.binary_repr(delay_value, 4))
    command[-17:-14] = list(np.binary_repr(refractory_max, 3))
    command[-14:-13] = list(np.binary_repr(int(dual_synapse_en), 1))
    command[-13:-9]  = list(np.binary_repr(shadow_uram_offset, 4))
    command[-8:-7]   = list(np.binary_repr(int(legacy_noise_en), 1))
```

## IEP Parameter Read (internal_events_processor.v)

```verilog
always @(posedge clk) begin
    if (ci2iep_dout[53] == 1'b1 && !ci2iep_empty) begin  // parameter write
        leak_param             <= dout_neuron_param_mem[83:78];
        shift_param            <= dout_neuron_param_mem[77:72];
        exec_neuron_model_param <= dout_neuron_param_mem[71:70];
        threshold_param        <= dout_neuron_param_mem[69:34];
        boundary               <= dout_neuron_param_mem[33:21];
        delay_value_param      <= dout_neuron_param_mem[20:17];
        refractory_max_param   <= dout_neuron_param_mem[16:14];
        dual_synapse_en_param  <= dout_neuron_param_mem[13];
        shadow_uram_offset_param <= dout_neuron_param_mem[12:9];
        soft_reset_en_param    <= dout_neuron_param_mem[8];
        legacy_noise_en_param  <= dout_neuron_param_mem[7];
    end
end
```

## DVS Model Configuration

The DVS pickle `DVS_model_config_shift=0.pkl` stores shift=0. At runtime, the test converts this:

```python
for key in connections:
    neuron_obj = connections[key][1]
    if hasattr(neuron_obj, 'shift') and neuron_obj.shift == 0:
        neuron_obj.shift = -17          # L6d: no noise
    neuron_obj.legacy_noise_en = 1       # Enable 35-bit MP mode
```

This ensures the DVS model runs in 2024-compatible 35-bit mode while basic tests use default 32-bit L6d mode.
