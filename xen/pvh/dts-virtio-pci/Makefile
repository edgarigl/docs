
pt.dtb: pt.dts pcie-host-bridge-irqx3.dtsi

%.dtb: %.dts
	$(CC) -E -nostdinc ${CPPFLAGS} -x assembler-with-cpp -MD -o $(@:.dtb=.dts.i) $<
	dtc -I dts -O dtb -o $@ $(@:.dtb=.dts.i)

