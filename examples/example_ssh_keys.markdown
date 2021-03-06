## Distribute ssh keys

In this example we assume that we have collected all users' public keys into a single source area on the server and that users exist on the clients (and have corresponding home directory). The specifics of SSH configuration has to be handled elsewhere. First we copy the keys we need to localhost, and then edit them into the the user's local keyring. 

You have to adapt this policy in the mentioned places for it to work in your environment.

	bundle agent distribute_ssh_keys
	{
	vars:

	    "users"             slist => { "user1", "user2", ... };   # List of users to be included in key distribution.
		                                                          # Modify to include actual users.
	    "source_server"    string => "$(sys.policy_hub)";         # Server where keys are stored
	    "source_directory" string => "/var/cfengine/masterfiles/ssh_keys"; # Source directory of key files
	    "local_cache"      string => "/var/cfengine/inputs/ssh_keys";      # Local cache of key files

	files:

	  # Local copy of keys for the policy server (*.pub files are not copied by default, only policy and text files)
	  am_policy_hub::

	   "$(local_cache)/$(users).pub"

		 comment => "Copy public keys from a an authorized source into a cache on localhost",
		   perms => mo("600","root"),
	       copy_from => local_cp("$(source_directory)/$(users).pub"),
		  action => if_elapsed("60");  # This ensures that checks for updates are only performed every 60 minutes

	  # Remote copy of keys for the clients (*.pub files are not copied by default, only policy and text files)
	  !am_policy_hub::

	   "$(local_cache)/$(users).pub"

		 comment => "Copy public keys from a an authorized source into a cache on localhost",
		   perms => mo("600","root"),
	       copy_from => remote_cp("$(source_directory)/$(users).pub","$(source_server)"),
		  action => if_elapsed("60");  # This ensures that checks for updates are only performed every 60 minutes

	  # Ensure that authorized_keys file exists and has permissions 600 and call a file editing promise
	  any::

	   "/home/$(users)/.ssh/authorized_keys"
		 comment => "Edit the authorized keys into the user's personal keyring",
		  create => "true",
		   perms => m("600"),
	       edit_line => insert_file_if_no_line_matching("$(users)","$(local_cache)/$(users).pub"),
		  action => if_elapsed("60");  # This ensures that edits are only performed every 60 minutes
	}

	#####

	bundle edit_line insert_file_if_no_line_matching(user,file)
	{

	# Check if user exists in the authorized_keys file
	classes:

	  "have_user" expression => regline("$(user).*","$(this.promiser)");

	# Insert the content of the key file into authorized_keys if the user's key is not already there
	insert_lines:

	  !have_user::

	    "$(file)"
		 insert_type => "file";
	}

You can find this bundle in the file <path/to/example_ssh_keys.cf TBD>

The following usage of this bundle presumes that you integrate it into the main policy file, `promises.cf`, and that the CFEngine standard library is included in `promises.cf`. To use this bundle:

1. Copy the above content into `/var/cfengine/masterfiles/example_ssh_keys.cf` or copy the file from `<path/to/example_ssh_keys.cf>` to `/var/cfengine/masterfiles`.

2. Insert the bundle name in the `bundlesequence` section of the main policy file (`/var/cfengine/masterfiles/promises.cf`) on the policy server:

	    bundlesequence => {
		                    ...
		                    "distribute_ssh_keys",
		                    ...
		                  };

3. Insert the policy file name in the `inputs` section of the main policy file (`/var/cfengine/masterfiles/promises.cf`) on the policy server:

            inputs => {
                        ...
                        "example_ssh_keys.cf",
                        ...
                      };

This policy snippet will now be executed every five minutes along with the rest of your main policy.

