# Makefile-HOWTO

This document will be used to explain what the various sections of a Makefile will have to look like, depending on the OPSI-package type.

This is the absolute minimum required in the header section:

    # HEADER
    OPSI_PACKAGE_NAME    = foobar.op
    OPSI_PACKAGE_VERSION = 1
    SOFTWARE_NAME        = FooBar
    SOFTWARE_VERSION     = 1.0
    TIMESTAMP            = $(shell date -R)
    MAINTAINER_NAME      = John Doe
    MAINTAINER_EMAIL     = maintainer@example.com
    OPSI_PACKAGE_DESC    = This is the $(SOFTWARE_NAME) package, packaged for DotOP by $(MAINTAINER_NAME) on $(TIMESTAMP). It does nothing.
    
    BUILD_DIR            = ./build
    TARGET               = $(OPSI_PACKAGE_NAME)_$(SOFTWARE_VERSION)-$(OPSI_PACKAGE_VERSION).opsi


This is the absolute minimum required in the action section:
Note: All indents in the *ACTION* section must be *TAB* characters, not four spaces, not eight spaces, TABs. This is essential. The make command *will* fail if you do not use TABs.


    # ACTION
    .PHONY: clean distclean
    
    all: clean build
    
    deploy: all clean install
    
    # Optional actions would expand the "all:" line to e.g. "all: clean someaction build someotheraction"
    
    $(BUILD_DIR)/$(TARGET): ./CLIENT_DATA ./OPSI/control # check that we have the minimum required files/dirs before starting the build
    	mkdir -p $(BUILD_DIR)
    	cp -alf ./CLIENT_DATA $(BUILD_DIR)
    	cp -af ./OPSI $(BUILD_DIR) # no hardlinking here, even though sed would be smart enough to break the link
    	# sed is called to fill the actual values from above into ./build/OPSI/control
    	sed -e "s/OPSI_PACKAGE_NAME/$(OPSI_PACKAGE_NAME)/g" \
    	    -e "s/OPSI_PACKAGE_VERSION/$(OPSI_PACKAGE_VERSION)/g" \
    	    -e "s/OPSI_PACKAGE_DESC/$(OPSI_PACKAGE_DESC)/g" \
    	    -e "s/SOFTWARE_NAME/$(SOFTWARE_NAME)/g" \
    	    -e "s/SOFTWARE_VERSION/$(SOFTWARE_VERSION)/g" \
    	    -e "s/MAINTAINER_NAME/$(MAINTAINER_NAME)/g" \
    	    -e "s/MAINTAINER_EMAIL/$(MAINTAINER_EMAIL)/g" \
    	    -e "s/TIMESTAMP/$(TIMESTAMP)/g" \
    	    -i $(BUILD_DIR)/OPSI/control

    # This is where a "someaction" block would go

    $(TARGET): $(BUILD_DIR)/$(TARGET) # tells make what to do to create the target
    	opsi-makeproductfile $(BUILD_DIR)

    build: $(TARGET) # requires an existing target, will be created according to recipe above if missing

    # This is where a "someotheraction" block would go

    $(TARGET).uploaded:
    	opsi-package-manager -i $(TARGET)
    	touch $(TARGET).uploaded

    install: $(TARGET).uploaded

    clean:
    	rm -rf $(BUILD_DIR)

    distclean: clean
    	rm -f $(TARGET) $(TARGET).uploaded

A Makefile that looks just like that, but without the verbose comments, is stored in the same directory as the file you are reading right now, named Makefile-001-minimum.
