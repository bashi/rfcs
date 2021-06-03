# RFC #72: Address space overrides

## Summary

The goal of this RFC is to enable web platform tests to cover the
[CORS-RFC1918](https://wicg.github.io/cors-rfc1918) specification entirely.
More specifically, to enable tests to exercise user agent behavior in the face
of responses served from `local`, `private` and `public`
[address spaces](https://wicg.github.io/cors-rfc1918#address-space).

This is achieved by passing command-line flags to the browser under test forcing
it to artificially consider specific IP addresses `local`, `private` or
`public`.

This RFC aims to fix
[web-platform-tests/wpt#26166](https://github.com/web-platform-tests/wpt/issues/26166).

## Details

### Address allocation

For the purposes of testing, declare that the following subnets are not `local`
but instead:

* 127.1.0.0/16: `private`
* 127.2.0.0/16: `public`

### Browser changes

A new configuration surface is added to each browser (it need not be uniform,
though that would certainly help with implementation) that allows overriding the
address space derived from specific IP addresses. For example, this could be a
command-line flag:

```sh
--test-override-address-space=127.1.0.0/16:private,127.2.0.0/16:public
```

Test-only code wires this command-line flag to the code in the browser under
test that determines the address space of a network endpoint.

### Test runner changes

The web platform test runner sets command-line flags on browsers such that the
browsers implement the custom address space mapping described above during
tests.

Corresponding entries are added to the hosts file, as generated by
[make_hosts_file()](https://github.com/web-platform-tests/wpt/blob/master/tools/serve/serve.py#L513-L530):

```
127.1.0.1 private.web-platform.test
127.2.0.1 public.web-platform.test
```

This allows web platform tests to exercise browser behavior in the face of
`private` and `public` IP addresses, simply by targeting the above domains.

## Risks

If the tests run on an IPv6-only system, the system might not know how to route
requests to the IPv4 loopback address range? This seems like an extremely remote
possibility, in which case many other assumptions made by the test framework
might have to be revisited anyway.

## Alternatives considered

### Override addresses themselves

This approach was suggested by @ddragana.

A similar configuration mechanism is exposed by browsers by which the test
runner can override some IP addresses, such that the browser under test will
consider that sockets connected to IP A are in fact connected to a fake IP B.

For example, as a command-line flag:

```
--test-ip-address-override=127.1.0.1:8.8.8.8,127.2.0.1:192.168.1.1
```

Would force the browser to consider sockets connected to `127.1.0.1` as instead
being connected to `8.8.8.8` and likewise for `127.2.0.1` to `192.168.1.1`.

This approach allows overriding the address space without introducing the notion
of address spaces to the configuration surface. @ddragana argues that it is less
failure-prone than the above proposal due to it being simpler to implement.

In Chromium, however, this approach likely would require plumbing the fake IP
address lower into the network stack than we would have to plumb a fake address
space. Thus it seems that instead this would be harder to implement correctly.

### Use individual IP addresses

Same as above, but allow overriding address spaces per IP address instead of
per subnet.

The marginal difficulty of overriding per subnet seems low, and provides plenty
of test IP addresses to exercise intra-address-space, cross-ip-address requests,
which might come in handy [in the
future](https://github.com/WICG/cors-rfc1918/pull/1#issuecomment-721110250).

### Use real IP addresses

If we need to test user agent behavior in the face of a variety of IP addresses,
then... just do it!

This approach would have the web platform test runner configure additional
loopback network devices and assign them `private` and `public` IP addresses.
Alternatively, the test runner could probably reroute traffic heading for these
IP addresses to `127.0.0.1` through some iptables magic. Then we could configure
the hosts file to resolve specific domains to those IP addresses, and test the
whole shebang end to end.

This approach's main pro is that it truly is an end to end test of the feature,
exercising production code to its full extent. By virtue of requiring no
test-only hooks, it also requires no work on the part of browser developers.

Its main con is that it requires the test runner to have system privileges. One
could design around that by restricting system privileges to the test setup
phase. That refactoring however would require a fair amount of work to ensure no
setup step is broken because of the IP address hijacking.

Additionally, it requires work to support for each test execution environment.

Finally, it arguably requires the purchase of a public IP address for our use.
If not, then we open ourselves up to esoteric failures due to the hijacking of a
legitimate IP address for test purposes, likely in a long enough time for
everyone to have forgotten the existence of this hack.

### Extend WebDriver

This alternative was initially recommended by @stephenmcgruer in
[web-platform-tests/wpt#26166](https://github.com/web-platform-tests/wpt/issues/26166).

The gist of this alternative is this: extend WebDriver to allow overriding
the address space derived from specific IP addresses or domains.

Its main pro is that such an approach would be standardized at a higher level
than web platform tests alone.

I reached out to some WebDriver experts to explore this avenue first. I was
informed that WebDriver's audience is web developers seeking to test their
creations rather than browser developers. As such, they recommended a simpler
approach with command-line flags instead.