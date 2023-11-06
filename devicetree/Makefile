OBJDIR = obj
NAME = rk3528-tvbox

.PHONY: all clean

all: $(NAME).dtb $(NAME).dtb.txt

PRESET = ORIG

CPPFLAGS_ORIG = \
	-DNO_DMC_EXTRA=1 \
	-DNO_DFI=1 \
	-DNO_MBIST_VMIN=1 \
	-DNO_SECURE_OTP=1 \
	-DNO_TSADC_TRIM=1 \
	-DNO_BT_SOUND=1 \
	-DOLD_USB_QUIRK=1 \
	-DLOW_CPU_OPP=1 \

CPPFLAGS_LINUX = \
	-DLIMA_COMPAT=1 \
	-DSPI_PINS_NONE=0 -DHDMI_IDLE=0 \
	-DI2S_PINS_MERGE=0 -DTVBOX_FIXES=1 \

CPPFLAGS = $(CPPFLAGS_$(PRESET))
DTCFLAGS = -q

clean:
	$(RM) -r $(OBJDIR)

$(OBJDIR):
	mkdir -p $(OBJDIR)

-include $(OBJDIR)/$(NAME).dts.d

$(OBJDIR)/%.dts.tmp: %.dts | $(OBJDIR)
	$(CC) -nostdinc -I. -undef -E -x assembler-with-cpp $(CPPFLAGS) -MMD -MP -MT $@ $< -o $@

$(NAME).dtb: $(OBJDIR)/$(NAME).dts.tmp
	dtc -I dts -O dtb $< -o $@ --symbols $(DTCFLAGS)

$(NAME).dtb.txt: $(NAME).dtb
	dtc -I dtb -O dts $< -o $@ $(DTCFLAGS)

