---
title: DirectX 11 Render Pipeline
portfolioCover: img/dx11-cover.png
weight: 2
---
Created December 2022

{{< youtube SnfcaVfUyRA >}}

# What is it?

- **Engine**: My own!
- **Language**: C++ and HLSL
- **Source Code:** [Here!](https://github.com/gecko-devel/directx11-renderer-uni2022)

This small application is a project developed for university that involves learning how the DirectX 11 render pipeline works and how to implement various lighting techniques.

Specifically, the following features have been implemented:

- Reading from YAML to add materials, cameras, scenery, and other settings without recompiling.
- Blinn-Phong shading, including ambient, diffuse, and specular lighting.
- Materials include albedo and specular map textures with lighting interaction.
- Point, Spot, and Directional lights, configurable via YAML.
- A free camera and normal camera, switchable using the number keys.
- Pixel colour blending for translucent objects
- Fog that also uses colour blending.

# How do the lights work?

One of the problems I had to solve was how to deal with different types of lights on the same shader. To do this, I made a struct that represents directional, point, and spotlights at the same time, using an enum to determine which light the struct contains data for. 

```cpp
enum LightType
{
	DIRECTIONAL_LIGHT = 0,
	POINT_LIGHT = 1,
	SPOT_LIGHT = 2
};

struct Light
{
public:
	XMFLOAT4 Color;
	// -------------- 16 bytes
	XMFLOAT3 Direction;
	LightType lightType;
	// -------------- 16 bytes
	XMFLOAT3 Position;
	FLOAT Attenuation;
	// -------------- 16 bytes
	FLOAT SpotAngle;
private:
	XMFLOAT3 _padding;
};
```

As the lights are configurable using a YAML file, they need to be read using the `yaml-cpp` library when the program loads. The lighting section of the config file looks like this:

```yml
lighting:
  ambientLight: [0.2, 0.2, 0.2, 1]

  directionalLights:
    - direction: [-0.5, 0.5, 0.1]
      color: [0.8, 0.8, 0.8, 1.0]

  pointLights:
    - position: [4.0, 3.0, -4.0]
      color: [0.0, 0.0, 1.0, 1.0]
      attenuation: .5

  spotLights:
    - position: [2, 5, 0]
      color: [0, 1, 0, 1]
      attenuation: 0.1
      direction: [-0.2, -1, 0]
      spotAngle: 10 # degrees

    - position: [-4, 5, 0]
      color: [1, 0, 0, 1]
      attenuation: 0.01
      direction: [0, -1, 0]
      spotAngle: 10 # degrees

	  [...]
```

The program then reads the YAML and loads the lights into memory. It also keeps track of how many lights in total are being loaded.

```cpp
void Application::LoadSceneFromConfig()
{
	[...]

    // Read lights
    int i = 0;

    for (YAML::Node dlNode : _config["lighting"]["directionalLights"])
    {
        Light light;
        light.lightType = DIRECTIONAL_LIGHT;

        light.Color = dlNode["color"].as<XMFLOAT4>();
        light.Direction = dlNode["direction"].as<XMFLOAT3>();

        _lights[i] = light;
        i++;
    }

    // Read point lights
    for (YAML::Node plNode : _config["lighting"]["pointLights"])
    {
        Light light;
        light.lightType = POINT_LIGHT;

        light.Color = plNode["color"].as<XMFLOAT4>();
        light.Position = plNode["position"].as<XMFLOAT3>();
        light.Attenuation = plNode["attenuation"].as<float>();

        _lights[i] = light;
        i++;
    }

    // Read spotlights
    for (YAML::Node slNode : _config["lighting"]["spotLights"])
    {
        Light light;
        light.lightType = SPOT_LIGHT;

        light.Color = slNode["color"].as<XMFLOAT4>();
        light.Position = slNode["position"].as<XMFLOAT3>();
        light.Attenuation = slNode["attenuation"].as<float>();
        light.Direction = slNode["direction"].as<XMFLOAT3>();
        light.SpotAngle = slNode["spotAngle"].as<float>();

        _lights[i] = light;
        i++;
    }
    _numLights = i;
}
```

When the rendering begins, it takes the information of those lights and passes it to the constant buffer, ready to be used by the shader:

```cpp
for (GameObject* go : _orderedGameObjects)
{
	// Set the world matrix
	XMMATRIX world = XMLoadFloat4x4(go->GetWorld());

	// Make a constant buffer template with new shader variable values
	ConstantBuffer cb;
	cb.World = XMMatrixTranspose(world);
	cb.View = XMMatrixTranspose(view);
	cb.Projection = XMMatrixTranspose(projection);

	cb.AmbientLight = _ambientLight;

	cb.fog = _fog;

	// copy lights to constant buffer
	std::copy(std::begin(_lights), &_lights[_numLights], std::begin(cb.lights));
	cb.numLights = _numLights;

	[...]

	// Set the constant buffer on each shader so it uses the piped in data
	_pImmediateContext->VSSetConstantBuffers(0, 1, &_pConstantBuffer);
	_pImmediateContext->PSSetConstantBuffers(0, 1, &_pConstantBuffer);        

	// DRAW!
	_pImmediateContext->DrawIndexed(go->GetMeshData()->IndexCount, 0, 0);
}
```

Now that the constant buffer contains the lights, we can finally use them inside the shader! To do this, I just use a switch-case statement to check the type of light in the array and calculate it accordingly.

This code is in HLSL:

```js
// Lights
for (int i = 0; i < numLights; i++)
{
	switch (lights[i].LightType)
	{
		case 0: // Directional Light
			{
				diffuse += Diffuse(lights[i], normalize(lights[i].Direction), input.NormalW);
				specular += Specular(lights[i], viewerDir, normalize(lights[i].Direction), input.NormalW, input.TexCoord);
			}
			break;
		
		case 1: // Point Light
			{
				float3 lightVector = lights[i].Position - input.PosW;
				float distanceToLight = length(lightVector);
				lightVector = normalize(lightVector);
		
				float attenuation = Attenuation(lights[i], distanceToLight);
			
				diffuse += Diffuse(lights[i], lightVector, input.NormalW) * attenuation;
				specular += Specular(lights[i], viewerDir, lightVector, input.NormalW, input.TexCoord) * attenuation;
			}
			break;
		
		case 2: // Spot Light
			{
				// Get direction of the light to the current pixel's world position
				float3 lightVector = lights[i].Position - input.PosW;
				float distanceToLight = length(lightVector);
				lightVector = normalize(lightVector);
	
				// Falloff as light vector gets close to maximum angle to normal
				// Most code is derived (not directly copied) from here: https://www.3dgep.com/texturing-lighting-directx-11/#Spotlight_Cone
				float minCos = cos(radians(lights[i].SpotAngle));
				float maxCos = (minCos + 1.0f) / 2.0f;
				float cosAngle = dot(normalize(lights[i].Direction), -lightVector);
				float spotIntensity = smoothstep(minCos, maxCos, cosAngle);
			
				float attenuation = Attenuation(lights[i], distanceToLight);
			
				diffuse += Diffuse(lights[i], lightVector, input.NormalW) * attenuation * spotIntensity;
				specular += Specular(lights[i], viewerDir, lightVector, input.NormalW, input.TexCoord) * attenuation * spotIntensity;
			}
			break;
	}

```

## Credits

- [yaml-cpp](https://github.com/jbeder/yaml-cpp) by Jesse Beder is licenced under the MIT licence.
- [Patchy Medow](https://freepbr.com/materials/patchy-meadow/) texture by freepbr.com
- [Hand Sculpture](https://skfb.ly/oxY6x) by re1monsen is licensed under [Creative Commons Attribution](http://creativecommons.org/licenses/by/4.0/)
- Base framework provided by the university, as seen in the first commit.

![Screenshot](/img/directx-sc1.png)

![Screenshot](/img/directx-sc2.png)

![Screenshot](/img/directx-sc3.png)

![Screenshot](/img/directx-sc4.png)