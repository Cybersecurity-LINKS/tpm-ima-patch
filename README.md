# rpi-5.15.y TPM2.0 & IMA patch

## Problem Statement

Integrity Measure Architecture (IMA) starts before the detection of Trusted Platform Module (TPM) device in Kernel v5.15.
Here is our .config solution that enable these modules to be built-in:

```
CONFIG_INTEGRITY=y
CONFIG_IMA=y
CONFIG_IMA_MEASURE_PCR_IDX=10
CONFIG_IMA_NG_TEMPLATE=y
CONFIG_IMA_DEFAULT_TEMPLATE="ima-ng"
CONFIG_IMA_DEFAULT_HASH_SHA256=y
CONFIG_IMA_DEFAULT_HASH="sha256"
CONFIG_IMA_AUDIT=y
CONFIG_IMA_LSM_RULES=y
CONFIG_TCG_TPM=y
CONFIG_HW_RANDOM_TPM=y
CONFIG_TCG_TIS_CORE=y
CONFIG_TCG_TIS_SPI=y
CONFIG_SPI_BCM2835=y
```
We have noticed that in the function `ima_init()` there is a call to `tpm_default_chip()` that should return a valid tpm instance, but it always fails to `"No TPM chip found, activating TPM-bypass!"`. In that function there is a call to `idr_get_next()` which returns NULL because the `idr_replace()` function in `tpm_add_char_device()` is called after the call to `ima_init()->tpm_default_chip()`.

We introduced some debugging prints in order to detect the function calls order (_Figure 1_). In the picture below (_Figure 2_), we can see that the `idr_replace()` is called at second 2.3 ("tpm: TPM available" print) which is after ima_init calls tpm_default_chip to retrieve the tpm device (which takes only 1.65 seconds). To recap, at second 1.65 IMA did not find any TPM because the TPM device registration/loading finishes later.

![Figure 1](https://user-images.githubusercontent.com/102968484/164282086-96f2398c-29c8-4e01-9114-45d2ce9f9c84.jpg)

![Figure 2](https://user-images.githubusercontent.com/102968484/164282156-b8262f73-80ac-4ed9-9d6c-e12fa45dbadf.jpg)

## Solution

We managed to get TPM finish before IMA looks for it by making the following changes.

1. First we postponed the bcm2835 clock driver initialization by substituting the `postcore_initcall(__bcm2835_clk_driver_init)` with `subsys_initcall(__bcm2835_clk_driver_init)`.

2. Second we forced the “TPM Driver for native SPI access” to run its probe routine _synchronously_ with driver and device registration when IMA is enabled. In this way we are sure that the TPM registration finishes before IMA starts looking for the TPM device.

```c
static struct spi_driver tpm_tis_spi_driver = {
	.driver = {
		.name = "tpm_tis_spi",
		.pm = &tpm_tis_pm,
		.of_match_table = of_match_ptr(of_tis_spi_match),
		.acpi_match_table = ACPI_PTR(acpi_tis_spi_match),
#ifdef CONFIG_IMA
		.probe_type = PROBE_FORCE_SYNCHRONOUS,
#else
		.probe_type = PROBE_PREFER_ASYNCHRONOUS,
#endif
	},
	.probe = tpm_tis_spi_driver_probe,
	.remove = tpm_tis_spi_remove,
	.id_table = tpm_tis_spi_id,
};
module_spi_driver(tpm_tis_spi_driver);

MODULE_DESCRIPTION("TPM Driver for native SPI access");
```

In the picture below (Figure 3) we show that IMA successfully found the TPM device and didn’t fall in bypass mode.

![Figure 3](https://user-images.githubusercontent.com/102968484/164282310-a22aa802-0dcc-4f98-ab98-97dae5dca762.jpg)

## Apply the patch

In order to apply the patch execute the following:
```sh
patch -p1 -d linux/ < tpm_prior-to_IMA.patch
```

Anyhow, the patch has been merged on the official linux RPI kernel and is available starting from commit: `aaba5263553b28540c8d5a923cadcb424d9b3d12`. Here it is possible to check the Pull Request made: https://github.com/raspberrypi/linux/pull/5003
